package com.yce.service;

import com.yce.constant.Constant;
import com.yce.constant.MessageCode;
import com.yce.daoMapper.InfantAnbunMapper;
import com.yce.dto.CommonListResponseDto;
import com.yce.dto.CommonRequestBaseDto;
import com.yce.dto.CommonResponse;
import com.yce.dto.UserInfoToken;
import com.yce.dto.gakuseki_db.GakusekiDbImportKbnInfoDto;
import com.yce.dto.infant.InfantAnbunDetailDto;
import com.yce.dto.infant.InfantAnbunReqInfoDto;
import com.yce.entities.InfantAnbunSettingInfoEntity;
import com.yce.service.common.CommonResponseFactory;
import com.yce.utils.EmptyUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;

/**
 * 幼児集計：按分率関連機能
 */
@Service
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Slf4j
@Transactional(rollbackFor=Throwable.class)
public class InfantAnbunService {

    @Autowired
    private InfantAnbunMapper infantAnbunMapper;

    @Autowired
    private CommonResponseFactory commonResponseFactory;
    
    @Autowired
    private GakusekiDbService gakusekiDbService;

    // リスト区分
    private final String LIST_FLG_AUTO       = "";  // 自動集計結果リスト（デフォルト）
    private final String LIST_FLG_REGISTERED = "1"; // 登録済みリスト

    // 学籍区分・学校区分
    private final String GAKUSEKI_KBN_ELEM = "1"; // 学籍区分：小学校
    private final String SCHOOL_KBN_ELEM   = "1"; // 学校区分：小学校

    // 按分率の小数桁数（学校別按分率設定の型 decimal(5,3) に合わせる）
    private final int ANBUN_RATE_SCALE = 3;
    // 按分率の計算途中の小数桁数（端数処理は最後に1回だけ行う）
    private final int ANBUN_RATE_CALC_SCALE = 10;

    /**
     * 按分率リスト
     * @param reqData
     * @return
     * @throws Throwable
     */
    public CommonResponse<CommonListResponseDto> getAnbunList(InfantAnbunReqInfoDto reqData) throws Throwable {

        Integer reqYear = reqData.getYear();
        // 未指定（null）は空文字と同じく自動集計結果リストとして扱う
        String reqListFlg = reqData.getList_flg() == null ? LIST_FLG_AUTO : reqData.getList_flg();

        // 入力チェック
        if( reqYear == null || reqYear < Constant.targetMinYear ){
            return this.commonResponseFactory.createAddMsg(MessageCode.VALIDATION_FAIL, "年度は" + Constant.targetMinYear + "以上を指定してください");
        }

        // 入力チェック
        if( ! LIST_FLG_AUTO.equals(reqListFlg) && ! LIST_FLG_REGISTERED.equals(reqListFlg) ){
            return this.commonResponseFactory.createAddMsg(MessageCode.VALIDATION_FAIL, "リスト区分は空文字または" + LIST_FLG_REGISTERED + "を指定してください");
        }

        List<InfantAnbunDetailDto> anbunList;
        if( LIST_FLG_REGISTERED.equals(reqListFlg) ){

            // 按分率 登録済みリスト取得
            anbunList = this.infantAnbunMapper.selectListAnbunRegistered(reqYear, SCHOOL_KBN_ELEM);

            // 按分率登録が未実行の場合は、info:nullで返す
            if( EmptyUtil.isEmpty(anbunList) ){
                return this.commonResponseFactory.warn(MessageCode.W_NOT_EXEC_ANBUN_REGISTER);
            }

        }else{

            // 按分率 自動集計結果リスト取得
            anbunList = this.selectListAnbunAutoCalc(reqYear);

            // 集計できない（学籍DB（小学生）または学区域情報が未登録）場合は、info:nullで返す
            if( EmptyUtil.isEmpty(anbunList) ){
                return this.commonResponseFactory.warn(MessageCode.W_NO_DATA_ANBUN_LIST);
            }
        }

        // ページネーションは行わず全件返却
        CommonListResponseDto<InfantAnbunDetailDto> res = new CommonListResponseDto<>();
        res.setReturn_list(anbunList);
        res.setPage_size(0);
        res.setCurrent_page(1);
        res.setTotal_size(anbunList.size());

        return commonResponseFactory.success(res);
    }

    /**
     * 按分率登録 実行情報取得
     * @param reqData
     * @return
     * @throws Throwable
     */
    public CommonResponse<InfantAnbunSettingInfoEntity> getAnbunInfo(CommonRequestBaseDto reqData) throws Throwable {

        // 入力チェック
        if( reqData.getYear() == null || reqData.getYear() < Constant.targetMinYear ){
            return this.commonResponseFactory.createAddMsg(MessageCode.VALIDATION_FAIL, "年度は" + Constant.targetMinYear + "以上を指定してください");
        }

        InfantAnbunSettingInfoEntity et = this.infantAnbunMapper.selectOneAnbunSettingInfo(reqData.getYear());

        if( et == null ){
            // 実行情報無しなら、info:nullを返してあげる（FEに託す※メッセージ内容は画面では利用しないけど記録用としてレスポンス）
            return this.commonResponseFactory.warn(MessageCode.W_NOT_EXEC_ANBUN_REGISTER);
        }

        return this.commonResponseFactory.success(et);
    }

    /**
     * 按分率登録
     * 自動集計結果を、対象年度の学校別按分率設定として登録する（対象年度のデータを洗替）
     * ※採用按分率（fix_anbun_rate）には当年按分率を登録する
     * @param reqData
     * @param uToken
     * @return
     * @throws Throwable
     */
    public CommonResponse<InfantAnbunSettingInfoEntity> registerAnbun(CommonRequestBaseDto reqData, UserInfoToken uToken) throws Throwable {

        Integer reqYear = reqData.getYear();

        // 入力チェック
        if( reqYear == null || reqYear < Constant.targetMinYear ){
            return this.commonResponseFactory.createAddMsg(MessageCode.VALIDATION_FAIL, "年度は" + Constant.targetMinYear + "以上を指定してください");
        }

        // 学籍DB（小学校）の全区域登録が終わってないと実行できません
        GakusekiDbImportKbnInfoDto gakusekiDbDt = this.gakusekiDbService.createImportKbnInfo(reqYear, "1");
        if( ! "1".equals(gakusekiDbDt.getComplate_flg()) ){
            return this.commonResponseFactory.warn(MessageCode.W_NOT_COMPLETE_GAKUSEKI_DB_IMPORT);
        }
        
        // 按分率 自動集計結果リスト取得（登録時点の学籍DB・学区域情報で集計し直す）
        List<InfantAnbunDetailDto> anbunList = this.selectListAnbunAutoCalc(reqYear);

        // 集計できない（学籍DB（小学生）または学区域情報が未登録）場合は登録しない
        if( EmptyUtil.isEmpty(anbunList) ){
            return this.commonResponseFactory.warn(MessageCode.W_NO_DATA_ANBUN_LIST);
        }

        // 学校マスタに未登録の学校がある場合は登録しない（学校別按分率設定の学校コードは必須のため）
//        List<String> noMasterCodeList = anbunList.stream()
//                .flatMap(dto -> Stream.of(
//                        this.isNoSchoolMaster(dto.getShiteikou_code_pre(), dto.getShiteikou_code_gov()) ? dto.getShiteikou_code() : null,
//                        this.isNoSchoolMaster(dto.getUkeirekou_code_pre(), dto.getUkeirekou_code_gov()) ? dto.getUkeirekou_code() : null,
//                        // 受入校②は、受入校が2校の区域のみチェック
//                        this.hasUkeirekou2(dto) && this.isNoSchoolMaster(dto.getUkeirekou2_code_pre(), dto.getUkeirekou2_code_gov()) ? dto.getUkeirekou2_code() : null
//                ))
//                .filter(Objects::nonNull)
//                .distinct()
//                .sorted()
//                .toList();
//        if( EmptyUtil.isNotEmpty(noMasterCodeList) ){
//            return this.commonResponseFactory.createAddMsg(MessageCode.VALIDATION_FAIL,
//                    "学校マスタに学校コード（教育委員会・国指定）が登録されていない学校があります\n学校コード：" + String.join(",", noMasterCodeList));
//        }

        // 対象年度の学校別按分率設定を洗替
        this.infantAnbunMapper.deleteAnbunSetting(reqYear);

        for (int i = 0; i < anbunList.size(); i += Constant.BATCH_SIZE) {

            List<InfantAnbunDetailDto> batch = anbunList.subList(i, Math.min(i + Constant.BATCH_SIZE, anbunList.size()));

            //DBデータ作成
            this.infantAnbunMapper.insertListAnbunSetting(reqYear, batch);
        }

        // 実行情報更新
        this.infantAnbunMapper.updateInsertAnbunSettingInfo(reqYear, uToken);

        // レスポンス作成（更新後の実行情報）
        InfantAnbunSettingInfoEntity infoEt = this.infantAnbunMapper.selectOneAnbunSettingInfo(reqYear);

        return this.commonResponseFactory.success(infoEt);
    }

    /**
     * 按分率 自動集計結果リスト取得
     * 学籍DB・学区域情報から「指定校：受入校①：受入校②」ごとの学年別人数と過去3年分の採用按分率を取得し、
     * 当年按分率・直近3年平均按分率を計算して設定する（受入校②は、受入校が2校の区域のみ）
     * @param year 対象年度
     * @return 集計できない場合は空のリスト
     * @throws Throwable
     */
    private List<InfantAnbunDetailDto> selectListAnbunAutoCalc(Integer year) throws Throwable {

        List<InfantAnbunDetailDto> anbunList = this.infantAnbunMapper.selectListAnbunAuto(year, GAKUSEKI_KBN_ELEM, SCHOOL_KBN_ELEM);

        // 当年按分率・直近3年平均按分率を計算
        for( InfantAnbunDetailDto dto : anbunList ){

            // 1年生～6年生の人数（配列の添字0=1年生）
            Integer[] shiteiCnts = {
                    dto.getShiteikou_cnt_g1(), dto.getShiteikou_cnt_g2(), dto.getShiteikou_cnt_g3(),
                    dto.getShiteikou_cnt_g4(), dto.getShiteikou_cnt_g5(), dto.getShiteikou_cnt_g6()
            };
            Integer[] ukeireCnts = {
                    dto.getUkeirekou_cnt_g1(), dto.getUkeirekou_cnt_g2(), dto.getUkeirekou_cnt_g3(),
                    dto.getUkeirekou_cnt_g4(), dto.getUkeirekou_cnt_g5(), dto.getUkeirekou_cnt_g6()
            };
            // 受入校②なしの場合はすべてnull（=0人として扱う）
            Integer[] ukeire2Cnts = {
                    dto.getUkeirekou2_cnt_g1(), dto.getUkeirekou2_cnt_g2(), dto.getUkeirekou2_cnt_g3(),
                    dto.getUkeirekou2_cnt_g4(), dto.getUkeirekou2_cnt_g5(), dto.getUkeirekou2_cnt_g6()
            };

            // 受入校①
            dto.setAnbun_rate_this_year(this.calcAnbunRateThisYear(ukeireCnts, shiteiCnts, ukeireCnts, ukeire2Cnts));
            dto.setAnbun_rate_avg_3year(this.calcAnbunRateAvg(
                    dto.getAnbun_rate_this_year(),
                    dto.getAnbun_rate_1_year_ago(),
                    dto.getAnbun_rate_2_year_ago()
            ));

            // 受入校②（受入校が2校の区域のみ）
            if( this.hasUkeirekou2(dto) ){
                dto.setAnbun2_rate_this_year(this.calcAnbunRateThisYear(ukeire2Cnts, shiteiCnts, ukeireCnts, ukeire2Cnts));
                dto.setAnbun2_rate_avg_3year(this.calcAnbunRateAvg(
                        dto.getAnbun2_rate_this_year(),
                        dto.getAnbun2_rate_1_year_ago(),
                        dto.getAnbun2_rate_2_year_ago()
                ));
            }
        }

        return anbunList;
    }

    /**
     * 学校マスタ未登録（教育委員会コード・国指定コードのいずれかが未設定）か判定
     */
    private boolean isNoSchoolMaster(String schoolCodePre, String schoolCodeGov) {
        return EmptyUtil.isEmpty(schoolCodePre) || EmptyUtil.isEmpty(schoolCodeGov);
    }

    /**
     * 受入校②がある（1つの特調区域に受入校が2校ある）か判定
     */
    private boolean hasUkeirekou2(InfantAnbunDetailDto dto) {
        return EmptyUtil.isNotEmpty(dto.getUkeirekou2_code());
    }

    /**
     * 当年按分率を計算
     * 学年ごとに「対象の受入校の人数 /（指定校 + 受入校① + 受入校② の人数）」を求め、その平均を当年按分率とする
     * ※受入校が1校の場合は、受入校②の人数を0人として計算する（= 受入校の人数 /（指定校 + 受入校の人数））
     * ※全校0人の学年は平均の対象から除外する
     * ※全学年が対象外の場合はnull
     * @param targetCnts 按分率を求める受入校の学年別人数（1年生～6年生）
     * @param allCnts    分母となる全校（指定校・受入校①・受入校②）の学年別人数（1年生～6年生）
     */
    private BigDecimal calcAnbunRateThisYear(Integer[] targetCnts, Integer[]... allCnts) {

        BigDecimal[] gradeRates = new BigDecimal[targetCnts.length]; // 学年別按分率（対象外の学年はnull）
        for( int i = 0; i < targetCnts.length; i++ ){
            int targetCnt = this.nullToZero(targetCnts[i]);

            // 分母：全校の人数
            int totalCnt = 0;
            for( Integer[] cnts : allCnts ){
                totalCnt += this.nullToZero(cnts[i]);
            }

            // 全校0人の学年は対象外
            if( totalCnt == 0 ){
                continue;
            }

            gradeRates[i] = BigDecimal.valueOf(targetCnt)
                    .divide(BigDecimal.valueOf(totalCnt), ANBUN_RATE_CALC_SCALE, RoundingMode.HALF_UP);
        }

        return this.calcAnbunRateAvg(gradeRates);
    }

    /**
     * 人数がnullの場合は0人とする
     */
    private int nullToZero(Integer cnt) {
        return cnt == null ? 0 : cnt;
    }

    /**
     * 按分率の平均を計算（小数第3位で四捨五入）
     * ※nullの値は平均の対象から除外する（データがある値だけで平均）
     * ※すべてnullの場合はnull
     */
    private BigDecimal calcAnbunRateAvg(BigDecimal... rates) {

        List<BigDecimal> targetRates = Arrays.stream(rates)
                .filter(Objects::nonNull)
                .toList();

        if( targetRates.isEmpty() ){
            return null;
        }

        BigDecimal sum = targetRates.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
        return sum.divide(BigDecimal.valueOf(targetRates.size()), ANBUN_RATE_SCALE, RoundingMode.HALF_UP);
    }
}




<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.yce.daoMapper.InfantAnbunMapper">

    <!--
        按分率 自動集計結果リスト
        学籍DB（小学生）と学区域情報を突合し、「指定校：受入校」ごとの学年別人数を集計する
        ・学籍DBの shiteikou_code は「通っている学校」のため、住所を学区域情報の特調区域（tokucho_kbn = '1'）と突合し、
          通っている学校 = 指定校(shiteikou_code_1) → 指定校の人数、= 受入校①(tokucho_kou_code_1) → 受入校①の人数、
          = 受入校②(tokucho_kou_code_2) → 受入校②の人数 とする
        ・同じ「指定校：受入校①：受入校②」の特調区域が複数ある場合は1行に集約する
          ※受入校が1校の区域（受入校②なし）と2校の区域は、受入校①が同じでも別の行になる
        ・学校名、教育委員会コードは学校マスタから取得し、学校別按分率設定から過去3年分の採用按分率を取得する
        ※当年按分率・直近3年平均按分率はService側で計算する
    -->
    <select id="selectListAnbunAuto" resultType="com.yce.dto.infant.InfantAnbunDetailDto">
        SELECT
             t.shiteikou_code
            ,s1.school_nm AS shiteikou_nm
            ,t.shiteikou_cnt_g1
            ,t.shiteikou_cnt_g2
            ,t.shiteikou_cnt_g3
            ,t.shiteikou_cnt_g4
            ,t.shiteikou_cnt_g5
            ,t.shiteikou_cnt_g6
            ,t.shiteikou_cnt_g1 + t.shiteikou_cnt_g2 + t.shiteikou_cnt_g3
                + t.shiteikou_cnt_g4 + t.shiteikou_cnt_g5 + t.shiteikou_cnt_g6 AS shiteikou_cnt_total
            ,t.ukeirekou_code
            ,s2.school_nm AS ukeirekou_nm
            ,t.ukeirekou_cnt_g1
            ,t.ukeirekou_cnt_g2
            ,t.ukeirekou_cnt_g3
            ,t.ukeirekou_cnt_g4
            ,t.ukeirekou_cnt_g5
            ,t.ukeirekou_cnt_g6
            ,t.ukeirekou_cnt_g1 + t.ukeirekou_cnt_g2 + t.ukeirekou_cnt_g3
                + t.ukeirekou_cnt_g4 + t.ukeirekou_cnt_g5 + t.ukeirekou_cnt_g6 AS ukeirekou_cnt_total
            ,t.ukeirekou2_code
            ,s3.school_nm AS ukeirekou2_nm
            ,t.ukeirekou2_cnt_g1
            ,t.ukeirekou2_cnt_g2
            ,t.ukeirekou2_cnt_g3
            ,t.ukeirekou2_cnt_g4
            ,t.ukeirekou2_cnt_g5
            ,t.ukeirekou2_cnt_g6
            ,t.ukeirekou2_cnt_g1 + t.ukeirekou2_cnt_g2 + t.ukeirekou2_cnt_g3
                + t.ukeirekou2_cnt_g4 + t.ukeirekou2_cnt_g5 + t.ukeirekou2_cnt_g6 AS ukeirekou2_cnt_total  /* 受入校②なしはNULL */
            ,a.anbun_rate_1_year_ago
            ,a.anbun_rate_2_year_ago
            ,a.anbun_rate_3_year_ago
            ,a.anbun2_rate_1_year_ago
            ,a.anbun2_rate_2_year_ago
            ,a.anbun2_rate_3_year_ago
        FROM (
            /* 指定校：受入校①：受入校② ごとの学年別人数 */
            SELECT
                 sa.year
                ,sa.shiteikou_code_1   AS shiteikou_code
                ,sa.tokucho_kou_code_1 AS ukeirekou_code
                ,NULLIF(sa.tokucho_kou_code_2, '') AS ukeirekou2_code  /* 空文字は受入校②なしとして扱う */
                /* 指定校の学年別人数 */
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 1 THEN 1 ELSE 0 END) AS shiteikou_cnt_g1
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 2 THEN 1 ELSE 0 END) AS shiteikou_cnt_g2
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 3 THEN 1 ELSE 0 END) AS shiteikou_cnt_g3
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 4 THEN 1 ELSE 0 END) AS shiteikou_cnt_g4
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 5 THEN 1 ELSE 0 END) AS shiteikou_cnt_g5
                ,SUM(CASE WHEN g.shiteikou_code = sa.shiteikou_code_1 AND g.grade = 6 THEN 1 ELSE 0 END) AS shiteikou_cnt_g6
                /* 受入校の学年別人数 */
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 1 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g1
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 2 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g2
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 3 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g3
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 4 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g4
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 5 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g5
                ,SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_1 AND g.grade = 6 THEN 1 ELSE 0 END) AS ukeirekou_cnt_g6
                /* 受入校②の学年別人数（受入校②なしの場合はNULL） */
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 1 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g1
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 2 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g2
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 3 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g3
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 4 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g4
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 5 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g5
                ,IF(NULLIF(sa.tokucho_kou_code_2, '') IS NULL, NULL, SUM(CASE WHEN g.shiteikou_code = sa.tokucho_kou_code_2 AND g.grade = 6 THEN 1 ELSE 0 END)) AS ukeirekou2_cnt_g6
            FROM
                school_area_info_data AS sa
            LEFT JOIN gakuseki_db_data AS g
                ON  g.year         = sa.year
                AND g.gakuseki_kbn = #{gakuseki_kbn}   /* 小学生 */
                AND g.ward_code    = sa.ward_code
                AND g.town_code    = sa.town_code
                AND g.aza_code     = sa.aza_code
                AND CONCAT(g.banchi_code_1, g.banchi_code_2, g.banchi_code_3, g.banchi_code_4)
                    BETWEEN sa.banchi_code_from_fix AND sa.banchi_code_to_fix
                AND g.shiteikou_code IN (sa.shiteikou_code_1, sa.tokucho_kou_code_1, sa.tokucho_kou_code_2)  /* 指定校 or 受入校①② に通う児童のみ */
                AND IFNULL(g.area_kbn, '') != '不'     /* 区域区分「不」は除外 */
            WHERE 1 = 1
                AND sa.year        = #{year}
                AND sa.tokucho_kbn = '1'               /* 特調区域のみ */
            GROUP BY
                 sa.year
                ,sa.shiteikou_code_1
                ,sa.tokucho_kou_code_1
                ,NULLIF(sa.tokucho_kou_code_2, '')
        ) AS t

        /* 学校コード（指定校用） */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMaster" />
        ) AS s1
            ON  s1.year        = t.year
            AND s1.school_code = t.shiteikou_code

        /* 学校コード（受入校①用） */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMaster" />
        ) AS s2
            ON  s2.year        = t.year
            AND s2.school_code = t.ukeirekou_code

        /* 学校コード（受入校②用） */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMaster" />
        ) AS s3
            ON  s3.year        = t.year
            AND s3.school_code = t.ukeirekou2_code

        /* 過去3年分の採用按分率（学校別按分率設定） */
        LEFT JOIN (
            <include refid="sqlPastAnbunRate" />
        ) AS a
            ON  a.shitei_school_code  =   s1.school_code
            AND a.ukeire_school_code  =   s2.school_code
            <![CDATA[ AND a.ukeire2_school_code <=> s3.school_code ]]>  /*受入校②なし（NULL）同士も一致させる */
            /* 受入校②が学校マスタ未登録（s3がNULL）の場合に、受入校1校の組み合わせと一致させないため */
            AND (t.ukeirekou2_code IS NULL) = (a.ukeire2_school_code IS NULL)

        ORDER BY
             t.shiteikou_code
            ,t.ukeirekou_code
            ,t.ukeirekou2_code   /* 受入校②なし（null）が先頭 */
    </select>

    <!--
        按分率 登録済みリスト
        按分率登録で登録した学校別按分率設定（対象年度）を取得する
        ・当年按分率 = 採用按分率（fix_anbun_rate）、直近3年平均按分率 = 登録時点の3年平均（ave_3year_anbun_rate）
          受入校②も同様（fix_anbun_rate_2、ave_3year_anbun_rate_2）
        ・学校コード（5桁）・学校名は、登録済みの教育委員会コード（PRE）で学校マスタから取得する
        ・過去3年分の採用按分率は、自動集計結果リストと同じく学校別按分率設定から取得する
    -->
    <select id="selectListAnbunRegistered" resultType="com.yce.dto.infant.InfantAnbunDetailDto">
        SELECT
             s1.school_code             AS shiteikou_code
            ,s1.school_nm               AS shiteikou_nm
            ,st.shitei_count_grade_1    AS shiteikou_cnt_g1
            ,st.shitei_count_grade_2    AS shiteikou_cnt_g2
            ,st.shitei_count_grade_3    AS shiteikou_cnt_g3
            ,st.shitei_count_grade_4    AS shiteikou_cnt_g4
            ,st.shitei_count_grade_5    AS shiteikou_cnt_g5
            ,st.shitei_count_grade_6    AS shiteikou_cnt_g6
            ,st.shitei_count_total      AS shiteikou_cnt_total
            ,s2.school_code             AS ukeirekou_code
            ,s2.school_nm               AS ukeirekou_nm
            ,st.ukeire_count_grade_1    AS ukeirekou_cnt_g1
            ,st.ukeire_count_grade_2    AS ukeirekou_cnt_g2
            ,st.ukeire_count_grade_3    AS ukeirekou_cnt_g3
            ,st.ukeire_count_grade_4    AS ukeirekou_cnt_g4
            ,st.ukeire_count_grade_5    AS ukeirekou_cnt_g5
            ,st.ukeire_count_grade_6    AS ukeirekou_cnt_g6
            ,st.ukeire_count_total      AS ukeirekou_cnt_total
            ,s3.school_code             AS ukeirekou2_code
            ,s3.school_nm               AS ukeirekou2_nm
            ,st.ukeire2_count_grade_1   AS ukeirekou2_cnt_g1
            ,st.ukeire2_count_grade_2   AS ukeirekou2_cnt_g2
            ,st.ukeire2_count_grade_3   AS ukeirekou2_cnt_g3
            ,st.ukeire2_count_grade_4   AS ukeirekou2_cnt_g4
            ,st.ukeire2_count_grade_5   AS ukeirekou2_cnt_g5
            ,st.ukeire2_count_grade_6   AS ukeirekou2_cnt_g6
            ,st.ukeire2_count_total     AS ukeirekou2_cnt_total
            ,st.fix_anbun_rate          AS anbun_rate_this_year
            ,st.ave_3year_anbun_rate    AS anbun_rate_avg_3year
            ,a.anbun_rate_1_year_ago
            ,a.anbun_rate_2_year_ago
            ,a.anbun_rate_3_year_ago
            ,st.fix_anbun_rate_2        AS anbun2_rate_this_year
            ,st.ave_3year_anbun_rate_2  AS anbun2_rate_avg_3year
            ,a.anbun2_rate_1_year_ago
            ,a.anbun2_rate_2_year_ago
            ,a.anbun2_rate_3_year_ago
        FROM
            infant_anbun_setting AS st

        /* 指定校の学校コード（5桁）・学校名 */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMasterByPre" />
        ) AS s1
            ON  s1.year            = st.year
            AND s1.school_code = st.shitei_school_code

        /* 受入校の学校コード（5桁）・学校名 */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMasterByPre" />
        ) AS s2
            ON  s2.year            = st.year
            AND s2.school_code = st.ukeire_school_code

        /* 受入校②の学校コード（5桁）・学校名 */
        LEFT JOIN (
            <include refid="sqlSchoolCodeMasterByPre" />
        ) AS s3
            ON  s3.year            = st.year
            AND s3.school_code = st.ukeire2_school_code

        /* 過去3年分の採用按分率（学校別按分率設定） */
        LEFT JOIN (
            <include refid="sqlPastAnbunRate" />
        ) AS a
            ON  a.shitei_school_code  =   st.shitei_school_code
            AND a.ukeire_school_code  =   st.ukeire_school_code
            <![CDATA[ AND a.ukeire2_school_code <=> st.ukeire2_school_code ]]>  /*受入校②なし（NULL）同士も一致させる */

        WHERE
            st.year = #{year}

        ORDER BY
             shiteikou_code
            ,ukeirekou_code
            ,ukeirekou2_code   /* 受入校②なし（null）が先頭 */
    </select>

    <!-- 学校別按分率設定 対象年度のデータを削除（登録時の洗替用） -->
    <delete id="deleteAnbunSetting">
        DELETE FROM
            infant_anbun_setting
        WHERE
            year = #{year}
    </delete>

    <!-- 学校別按分率設定 バルクインサート -->
    <insert id="insertListAnbunSetting">
        INSERT INTO infant_anbun_setting
        (
             infant_anbun_setting_id
            ,year
            ,shitei_school_code
            ,ukeire_school_code
            ,fix_anbun_rate
            ,ave_3year_anbun_rate
            ,shitei_count_grade_1
            ,shitei_count_grade_2
            ,shitei_count_grade_3
            ,shitei_count_grade_4
            ,shitei_count_grade_5
            ,shitei_count_grade_6
            ,shitei_count_total
            ,ukeire_count_grade_1
            ,ukeire_count_grade_2
            ,ukeire_count_grade_3
            ,ukeire_count_grade_4
            ,ukeire_count_grade_5
            ,ukeire_count_grade_6
            ,ukeire_count_total
            ,ukeire2_school_code
            ,fix_anbun_rate_2
            ,ave_3year_anbun_rate_2
            ,ukeire2_count_grade_1
            ,ukeire2_count_grade_2
            ,ukeire2_count_grade_3
            ,ukeire2_count_grade_4
            ,ukeire2_count_grade_5
            ,ukeire2_count_grade_6
            ,ukeire2_count_total
        )
        VALUES
        <foreach collection="dataList" item="data" separator="," >
            (
             UUID()
            ,#{year}
            ,#{data.shiteikou_code}
            ,#{data.ukeirekou_code}
            ,#{data.anbun_rate_this_year}   /* 採用按分率 = 当年按分率 */
            ,#{data.anbun_rate_avg_3year}
            ,#{data.shiteikou_cnt_g1}
            ,#{data.shiteikou_cnt_g2}
            ,#{data.shiteikou_cnt_g3}
            ,#{data.shiteikou_cnt_g4}
            ,#{data.shiteikou_cnt_g5}
            ,#{data.shiteikou_cnt_g6}
            ,#{data.shiteikou_cnt_total}
            ,#{data.ukeirekou_cnt_g1}
            ,#{data.ukeirekou_cnt_g2}
            ,#{data.ukeirekou_cnt_g3}
            ,#{data.ukeirekou_cnt_g4}
            ,#{data.ukeirekou_cnt_g5}
            ,#{data.ukeirekou_cnt_g6}
            ,#{data.ukeirekou_cnt_total}
            /* 受入校②（受入校が1校の場合はすべてNULL） */
            ,#{data.ukeirekou2_code}
            ,#{data.anbun2_rate_this_year}  /* 採用按分率 = 当年按分率 */
            ,#{data.anbun2_rate_avg_3year}
            ,#{data.ukeirekou2_cnt_g1}
            ,#{data.ukeirekou2_cnt_g2}
            ,#{data.ukeirekou2_cnt_g3}
            ,#{data.ukeirekou2_cnt_g4}
            ,#{data.ukeirekou2_cnt_g5}
            ,#{data.ukeirekou2_cnt_g6}
            ,#{data.ukeirekou2_cnt_total}
            )
        </foreach>
    </insert>

    <!-- 按分率登録 実行情報取得 -->
    <select id="selectOneAnbunSettingInfo" resultType="com.yce.entities.InfantAnbunSettingInfoEntity">
        SELECT
             year
            ,update_user_id
            ,update_user_nm
            ,update_date
        FROM
            infant_anbun_setting_info
        WHERE 1 = 1
            AND year = #{year}
        LIMIT 1
    </select>

    <!-- 按分率登録 実行情報 登録/更新 -->
    <insert id="updateInsertAnbunSettingInfo">
        INSERT INTO
            infant_anbun_setting_info
        (
             year
            ,regist_user_id
            ,regist_user_nm
            ,regist_event_id
            ,regist_date
            ,update_user_id
            ,update_user_nm
            ,update_event_id
            ,update_date
        ) VALUES (
             #{year}
            ,#{uToken.user_id}
            ,#{uToken.user_nm}
            ,#{uToken.event_id}
            ,CURRENT_TIMESTAMP()
            ,#{uToken.user_id}
            ,#{uToken.user_nm}
            ,#{uToken.event_id}
            ,CURRENT_TIMESTAMP()
        )
        ON DUPLICATE KEY UPDATE
             update_date     = VALUES(update_date)
            ,update_user_id  = VALUES(update_user_id)
            ,update_user_nm  = VALUES(update_user_nm)
            ,update_event_id = VALUES(update_event_id)
    </insert>

    <sql id="sqlPastAnbunRate">
        <!--
        過去3年分（対象年度の1年前～3年前）の採用按分率
        「指定校：受入校①：受入校②」（教育委員会コード）ごとに1行（受入校1校の場合、受入校②はNULL）
        ※同じ年度・組み合わせが複数行あっても1行に集約されるように集計関数で取得
        -->
        SELECT
             shitei_school_code
            ,ukeire_school_code
            ,ukeire2_school_code
            ,MAX(CASE WHEN year = #{year} - 1 THEN fix_anbun_rate END) AS anbun_rate_1_year_ago
            ,MAX(CASE WHEN year = #{year} - 2 THEN fix_anbun_rate END) AS anbun_rate_2_year_ago
            ,MAX(CASE WHEN year = #{year} - 3 THEN fix_anbun_rate END) AS anbun_rate_3_year_ago
            ,MAX(CASE WHEN year = #{year} - 1 THEN fix_anbun_rate_2 END) AS anbun2_rate_1_year_ago
            ,MAX(CASE WHEN year = #{year} - 2 THEN fix_anbun_rate_2 END) AS anbun2_rate_2_year_ago
            ,MAX(CASE WHEN year = #{year} - 3 THEN fix_anbun_rate_2 END) AS anbun2_rate_3_year_ago
        FROM
            infant_anbun_setting
        WHERE
            year BETWEEN #{year} - 3 AND #{year} - 1
        GROUP BY
             shitei_school_code
            ,ukeire_school_code
            ,ukeire2_school_code
    </sql>

    <sql id="sqlSchoolCodeMasterByPre">
        <!--
        学校マスタ（小学校）を教育委員会コード（PRE）単位にしたもの（登録済みリスト用）
        学校コードは自動集計結果リストと同じく、school_code_cityを5桁0埋めした値とする
        ※同一年度・同一教育委員会コードが複数行あっても1行に集約しておく
        -->
        SELECT
             year
            ,school_code_city AS school_code
            ,MIN(school_nm)                      AS school_nm
        FROM
            m_school_code
        WHERE
            school_kbn = #{school_kbn}
        GROUP BY
             year
            ,school_code_city
    </sql>

    <sql id="sqlSchoolCodeMaster">
        <!--
        学校マスタ（小学校）
        学区域情報・学籍DBの学校コード（5桁）と突合できるように、school_code_cityを5桁0埋めした値を学校コードとする
        ※集計後に結合するため、同一年度・同一学校コードが複数行あっても1行に集約しておく（人数が倍にならないように）
        -->
        SELECT
             year
            ,school_code_city AS school_code
            ,MIN(school_nm)                 AS school_nm
        FROM
            m_school_code
        WHERE
            school_kbn = #{school_kbn}
        GROUP BY
             year
            ,school_code_city
    </sql>

</mapper>
