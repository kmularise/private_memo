
* 이게 오히려 통상적으로 말하는 전표 규칙 같은 느낌?

각각의 계정 명이 어디에 차 대 중 어디에 와야 하는지?
-- auto-generated definition
create table tsol_cr_asj_ctl
(
    ist_cd       varchar(3)                                       not null,
    asj_tbl_cd   varchar(10)                                      not null,
    asj_cd       varchar(10)                                      not null,
    dr_cds_tc    varchar(1),
    asj_nm       varchar(200),
    asj_abv_nm   varchar(200),
    asj_tc       varchar(1),
    stm_ats_tc   varchar(1),
    del_yn       varchar(1)  default 'N'::character varying       not null,
    fst_enr_usid varchar(14) default '.'::character varying       not null,
    fst_enr_dtm  timestamp   default CURRENT_TIMESTAMP            not null,
    fst_enr_trid varchar(30) default '.'::character varying       not null,
    fst_enr_ip   varchar(15) default '0.0.0.0'::character varying not null,
    lst_chg_usid varchar(14) default '.'::character varying       not null,
    lst_chg_dtm  timestamp   default CURRENT_TIMESTAMP            not null,
    lst_chg_trid varchar(30) default '.'::character varying       not null,
    lst_chg_ip   varchar(15) default '0.0.0.0'::character varying not null,
    primary key (ist_cd, asj_cd, asj_tbl_cd)
);

comment on column tsol_cr_asj_ctl.ist_cd is '기관코드';

comment on column tsol_cr_asj_ctl.asj_tbl_cd is '계정과목표코드';

comment on column tsol_cr_asj_ctl.asj_cd is '계정과목코드';

comment on column tsol_cr_asj_ctl.dr_cds_tc is '차변대변구분코드';

comment on column tsol_cr_asj_ctl.asj_nm is '계정과목명';

comment on column tsol_cr_asj_ctl.asj_abv_nm is '계정과목약어명';

comment on column tsol_cr_asj_ctl.asj_tc is '계정과목구분코드';

comment on column tsol_cr_asj_ctl.stm_ats_tc is '결제계정구분코드';

comment on column tsol_cr_asj_ctl.del_yn is '삭제여부';

comment on column tsol_cr_asj_ctl.fst_enr_usid is '최초등록사용자ID';

comment on column tsol_cr_asj_ctl.fst_enr_dtm is '최초등록일시';

comment on column tsol_cr_asj_ctl.fst_enr_trid is '최초등록거래ID';

comment on column tsol_cr_asj_ctl.fst_enr_ip is '최초등록IP';

comment on column tsol_cr_asj_ctl.lst_chg_usid is '최종변경사용자ID';

comment on column tsol_cr_asj_ctl.lst_chg_dtm is '최종변경일시';

comment on column tsol_cr_asj_ctl.lst_chg_trid is '최종변경거래ID';

comment on column tsol_cr_asj_ctl.lst_chg_ip is '최종변경IP';

alter table tsol_cr_asj_ctl
    owner to postgres;