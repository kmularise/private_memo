
* 포지션처리유형 ID가 뭐임
* 

```sql
-- auto-generated definition
create table tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs
(
    ist_cd       varchar(3)                                       not null,
    asj_tbl_cd   varchar(10)                                      not null,
    pt_pcs_tp_id varchar(20)                                      not null,
    evl_area_id  varchar(20)                                      not null,
    slp_rul_id   varchar(5),
    del_yn       varchar(1)  default 'N'::character varying       not null,
    fst_enr_usid varchar(14) default '.'::character varying       not null,
    fst_enr_dtm  timestamp   default CURRENT_TIMESTAMP            not null,
    fst_enr_trid varchar(30) default '.'::character varying       not null,
    fst_enr_ip   varchar(15) default '0.0.0.0'::character varying not null,
    lst_chg_usid varchar(14) default '.'::character varying       not null,
    lst_chg_dtm  timestamp   default CURRENT_TIMESTAMP            not null,
    lst_chg_trid varchar(30) default '.'::character varying       not null,
    lst_chg_ip   varchar(15) default '0.0.0.0'::character varying not null,
    primary key (pt_pcs_tp_id, evl_area_id, asj_tbl_cd, ist_cd)
);

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.ist_cd is '기관코드';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.asj_tbl_cd is '계정과목표코드';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.pt_pcs_tp_id is '포지션처리유형ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.evl_area_id is '평가영역ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.slp_rul_id is '전표규칙ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.del_yn is '삭제여부';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.fst_enr_usid is '최초등록사용자ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.fst_enr_dtm is '최초등록일시';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.fst_enr_trid is '최초등록거래ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.fst_enr_ip is '최초등록IP';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.lst_chg_usid is '최종변경사용자ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.lst_chg_dtm is '최종변경일시';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.lst_chg_trid is '최종변경거래ID';

comment on column tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs.lst_chg_ip is '최종변경IP';

alter table tsol_cr_slp_rul_pt_pcs_tp_mpn_sfs
    owner to postgres;


```