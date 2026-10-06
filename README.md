# facts
  METHOD zif_health_check_etmp_scan~compare_facts.
************************************************************************
* Author      : Hemant Joshi  CG ID: CG21811
* Date        : 20.01.2026
* Reference   : ARDK940543
* Transport   : ARDK940543 / 13951
* FS No       : D2008
* Description : Compare facts from tables with Payload data.
************************************************************************
* Author      : Hemant Joshi  CG ID: CG21811
* Date        : 27.05.2026
* Reference   : ARDK943903
* Transport/RT: ARDK943903 / 13951
* FS No       : D2008
* Description : Storing address errors in another internal table
************************************************************************
* Author      : Elizabeth Blackburn      Net Id : CG22543
* Date        : 21.08.2026
* Reference   : ARXK960116
* Transport/RT: ARXK960116 /
* FS No       : D2008
* Description : Obtain output error messages from utility class method
************************************************************************
* Author      : Elizabeth Blackburn      Net Id : CG22543
* Date        : 25.08.2026
* Reference   : ARXK960116
* Transport/RT: ARXK960116 /
* FS No       : D2008
* Description : Adding fact map 'mapping type = object reference' scenario
************************************************************************
* Author      : Elizabeth Blackburn      Net Id : CG22543
* Date        : 02.09.2026
* Reference   : ARXK960116
* Transport/RT: ARXK960116 /
* FS No       : D2008
* Description : Adding fact map 'mapping type = structure' scenario
************************************************************************
* Author      : Elizabeth Blackburn      Net Id : CG22543
* Date        : 16.09.2026
* Reference   : ARXK960116
* Transport/RT: ARXK960116 /
* FS No       : D2008
* Description : Change instances of MAPPING_PI to MAPPING_VALUE due to
*               field deletion in ZTHEALTH_FCT_MAP
************************************************************************
* Author      : Elizabeth Blackburn      Net Id : CG22543
* Date        : 17.09.2026
* Reference   : ARXK960116
* Transport/RT: ARXK960116 /
* FS No       : D2008
* Description : Removing methods PROCESS_ITSB/ITSC/ITSF
************************************************************************

    CONSTANTS: lc_itsb TYPE ps_fact_type VALUE 'ITSB',
               lc_itsc TYPE ps_fact_type VALUE 'ITSC',
               lc_itsf TYPE ps_fact_type VALUE 'ITSF',
               lc_no   TYPE char1 VALUE 'N',
               lc_type TYPE vktyp_kk VALUE '25'.

    DATA : lt_fct_err      TYPE ty_zthealth_fct_err_t,
           lt_fct_err_adr  TYPE ty_zthealth_fct_err_t,    "ARDK943903
           ls_input1       TYPE zsxx_etmp_transaction59,
           lv_pay_fct_val  TYPE string,
           lv_partner      TYPE char17,
*           lv_path         TYPE zmapping,
           lv_item_no      TYPE i VALUE 0,
           lv_first_time   TYPE char1,
           lv_error        TYPE boole_d ##NEEDED,
           lv_pay_fct_val2 TYPE ps_fact_generic,
           lv_psobkey      TYPE psobkey_ps.

    FIELD-SYMBOLS:      <fs_fact_fld> TYPE any. "ARXK960116

    CLEAR : et_fct_err, et_stat_events, ev_fact_error_msg.

*--> Fill the input structure from PROXY to ZIT6 FB generic Structure
    ls_input1      =  is_payload.

**    CALL METHOD zsxx_cl_ifis_01_safe_create_in=>fill_input_zit6
*
*      EXPORTING
*        is_input      = ls_input1
*      IMPORTING
*        es_zit6_input = ls_input1.
            CALL METHOD me->fill_zit6_input
              EXPORTING
                is_input      = ls_input1
                it_data       = IT_etmp_facts
              IMPORTING
                es_zit6_input = ls_input1.



    LOOP AT it_fact_map ASSIGNING FIELD-SYMBOL(<fs_map>) WHERE regime     = is_healthcheck_process-regime
                                                           AND process    = is_healthcheck_process-process
                                                           AND event_name = is_healthcheck_process-event_name
                                                           AND fbtyp      = is_healthcheck_process-fbtyp .

* compare address first, it does not have a mapping path
      IF lv_first_time IS INITIAL AND <fs_map>-fact_type = lc_itsb.
        lv_first_time = lc_no.
        READ TABLE it_etmp_facts INTO DATA(ls_facts) WITH KEY   fact_set       = <fs_map>-fact_set
                                                                fact_type      = <fs_map>-fact_type
                                                                fact_category  = <fs_map>-fact_category
                                                                fact_cat_seq   = <fs_map>-fact_cat_seq .
        IF sy-subrc = 0.
          lv_partner = ls_facts-partner.
          CALL METHOD me->check_address
            EXPORTING
              iv_partner = lv_partner
              iv_type    = ls_facts-fact_type
              is_payload = is_payload
              is_pay_log = is_pay_log
            IMPORTING
              et_fct_err = lt_fct_err_adr.    "ARDK943903

        ENDIF.
      ENDIF.
* following for records that do have a payload mapping path setup
      DATA(lt_etmp_facts) = it_etmp_facts.
      SORT lt_etmp_facts.
      LOOP AT lt_etmp_facts ASSIGNING FIELD-SYMBOL(<fs_etmp_facts>) WHERE  fact_set       = <fs_map>-fact_set
                                                                       AND fact_type      = <fs_map>-fact_type
                                                                       AND fact_category  = <fs_map>-fact_category
                                                                       AND fact_cat_seq   = <fs_map>-fact_cat_seq.
* Get ETMP Facts Data ( DFACTS )
        DATA(lv_etmp_fct_val) = <fs_etmp_facts>-value_generic .

* get period dates once per contract, used later in custom code
        IF lv_psobkey <> <fs_etmp_facts>-psobkey.
          lv_psobkey = <fs_etmp_facts>-psobkey.
          CLEAR gt_dpsob_bp_acc_per.
          CALL METHOD me->get_period_dates
            EXPORTING
              iv_psobkey          = lv_psobkey
              iv_partner          = <fs_etmp_facts>-partner
              iv_partneracctyp    = lc_type
              it_etmp_facts       = it_etmp_facts
              it_etmp_icr         = it_etmp_icr
            IMPORTING
              et_dpsob_bp_acc_per = gt_dpsob_bp_acc_per.
          SORT gt_dpsob_bp_acc_per BY persl abrzu.
        ENDIF.

*        lv_path = <fs_map>-mapping_value. "ARXK960116
        CASE <fs_map>-mapping_type.
*Begin of ARXK960116
          WHEN zif_health_check_etmp_scan~gc_map_structure_value.

            IF <fs_map>-mapping_value CS '['. "ARXK960116
              CALL METHOD me->get_value_from_deep_structure
                EXPORTING
                  is_payload = is_payload
                  iv_path    = <fs_map>-mapping_value "ARXK960116
                IMPORTING
                  ev_value   = lv_pay_fct_val.
            ELSE.
              ASSIGN COMPONENT <fs_map>-mapping_value OF STRUCTURE is_payload TO <fs_fact_fld>. "ARXK960116
              IF <fs_fact_fld> IS ASSIGNED.
                MOVE <fs_fact_fld> TO lv_pay_fct_val .
              ENDIF.
            ENDIF.

* CREATE_INCOME_SOURCE-BUSINESS_DETAILS[]-ACCOUNTING_PERIOD_START_DATE need to be calculated ARXK960116
*            IF <fs_map>-mapping_value =  'CREATE_INCOME_SOURCE-BUSINESS_DETAILS[]-ACCOUNTING_PERIOD_START_DATE'. "ARXK960116
*              DATA lv_date TYPE dats.
*              lv_date = lv_pay_fct_val.
*              CALL METHOD me->get_period_attributes
*                EXPORTING
*                  iv_periodicity = 'PQ'
*                  iv_date_from   = lv_date
*                IMPORTING
*                  ev_abrzu       = lv_date.
*              lv_pay_fct_val = lv_date.
*            ENDIF.
*            IF lv_pay_fct_val NE lv_etmp_fct_val.
*              lv_error = abap_true.
*            ENDIF.
* End of ARXK960116

          WHEN zif_health_check_etmp_scan~gc_map_fixed_value.
            IF  <fs_map>-fact_category = <fs_etmp_facts>-fact_category AND
                <fs_map>-fact_cat_seq = <fs_etmp_facts>-fact_cat_seq.
              MOVE <fs_map>-mapping_value TO lv_pay_fct_val .
            ENDIF.
          WHEN zif_health_check_etmp_scan~gc_map_store_custom_rule. "custom method
          CLEAR lv_pay_fct_val2.

            zif_health_check_etmp_scan~custom_rule_for_fact_compare( EXPORTING is_payload      = is_payload
                                                                               is_etmp_facts   = <fs_etmp_facts>
                                                                               is_pay_log      = is_pay_log
                                                                               is_fact_map     = <fs_map>
                                                                     IMPORTING ev_error        = lv_error
                                                                               ev_pay_fct_val  = lv_pay_fct_val2 ).
            lv_pay_fct_val = lv_pay_fct_val2.
* Begin of ARXK960116 - adding fact map 'mapping type = object reference' scenario
          WHEN zif_health_check_etmp_scan~gc_map_object_reference.
            ASSIGN (<fs_map>-mapping_value) TO  <fs_fact_fld>.
            IF <fs_fact_fld> IS ASSIGNED .
              MOVE <fs_fact_fld> TO lv_pay_fct_val .
            ENDIF.

            IF lv_pay_fct_val NE lv_etmp_fct_val.
              lv_error = abap_true.
            ENDIF.
* End of ARXK960116
*          WHEN OTHERS .
*            IF lv_path IS NOT INITIAL. "if Payload
*              CASE  <fs_etmp_facts>-fact_set.
*                WHEN  lc_itsb.
*
*                  CALL METHOD me->process_itsb
*                    EXPORTING
*                      is_map      = lv_path
*                      is_payload  = is_payload
*                    IMPORTING
*                      ev_fact_val = lv_pay_fct_val.
*                WHEN lc_itsc.
*                  CALL METHOD me->process_itsc
*                    EXPORTING
*                      is_map      = lv_path
*                      is_payload  = is_payload
*                    IMPORTING
*                      ev_fact_val = lv_pay_fct_val.
*
*                WHEN lc_itsf.
*                  CALL METHOD me->process_itsf
*                    EXPORTING
*                      is_map      = lv_path
*                      is_payload  = is_payload
*                    IMPORTING
*                      ev_fact_val = lv_pay_fct_val.
*
*              ENDCASE.
*            ENDIF.      "lv_path
        ENDCASE.

* Check if ETMP Fact value and Payload FACT Value ( post conversion  )  are correct
        IF lv_pay_fct_val NE lv_etmp_fct_val  .
* Populate stucture and add record
          lt_fct_err = VALUE #( BASE lt_fct_err ( submission_id   = is_pay_log-submission_id
                                                  seq_no          = is_pay_log-seqno
                                                  item_seq        = lv_item_no
                                                  contract_object = <fs_etmp_facts>-psobkey
                                                  fact_cat_seq    = <fs_map>-fact_cat_seq
                                                  fact_set        = <fs_map>-fact_set
                                                  fact_type       = <fs_map>-fact_type
                                                  fact_category   = <fs_map>-fact_category
                                                  etmp_fact_value = lv_etmp_fct_val
                                                  pay_fact_value  = lv_pay_fct_val
                                                  error_message   = zcl_itsa_health_check_utility=>get_fact_description( <fs_etmp_facts> ) "ARXK960116
                                                  ) ).

        ENDIF .
        CLEAR:  lv_pay_fct_val, lv_etmp_fct_val .
      ENDLOOP .
    ENDLOOP.

* if any errors,  add records to the reprocess table
*Start of Changes - ARDK943903
    IF lt_fct_err_adr IS NOT INITIAL.
      APPEND LINES OF lt_fct_err_adr TO lt_fct_err.
    ENDIF.
*End of Changes - ARDK943903
    IF lt_fct_err IS NOT INITIAL.
      me->check_date_to_fact( EXPORTING it_fct_err = lt_fct_err
                               IMPORTING et_fct_err = DATA(lt_upd_fct_err) ).

      IF lt_upd_fct_err IS NOT INITIAL.
* Start of Changes - ARDK943903
* resequence the item seq
        CLEAR lv_item_no.
        LOOP AT lt_upd_fct_err ASSIGNING FIELD-SYMBOL(<fs_upd_fct_err>).
          ADD 1 TO lv_item_no.
          <fs_upd_fct_err>-item_seq = lv_item_no.
        ENDLOOP.
*End of Changes - ARDK943903
        et_fct_err = lt_upd_fct_err.
        CONCATENATE gc_fbtyp gc_underscore gc_fact_error_message INTO ev_fact_error_msg.
      ENDIF.
    ENDIF.

  ENDMETHOD.
