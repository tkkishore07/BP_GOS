# BP_GOS
Workflow task attachment as BP GOS
The code explains on creating GOS attachment to the BP partner.
Initially when a file is attached to workitem task, an entry in SOFM is created and the corresponding file is saved in SOFM related tables as xstring format.
<img width="825" height="825" alt="image" src="https://github.com/user-attachments/assets/61c0834e-34b4-45aa-8652-9f5d5d843ee0" />

We can verify the same in SOM table
<img width="1129" height="517" alt="image" src="https://github.com/user-attachments/assets/9b250564-c012-4404-8555-3208bf0a0be9" />

To create an attachment as GOS for BP, we need to create an entry in table SRGBTBREL
We can achieve the same using bapi 'BINARY_RELATION_CREATE_COMMIT'.

Below is the code snippet, first we need to get the workitem of the attachment, then we need to read the attachment(SOFM entry) and then use the bapi


    DATA workitem_container TYPE REF TO cl_swf_cnt_container.
    FIELD-SYMBOLS <attachment> TYPE swf_utl_porb_tab.

    CHECK     workitem_id      IS NOT INITIAL
          AND business_partner IS NOT INITIAL.

    DATA(object) = VALUE borident( objkey  = business_partner
                                   objtype = zwf_businesspartner_govern_c=>businessobjects-businesspartner ).

    TRY.
        DATA(workitem_instance) = cl_swf_run_workitem_context=>get_instance( workitem_id ).
        DATA(container) = workitem_instance->if_wapi_workitem_context~get_wi_container( ).
        workitem_container ?= container.

        DATA(container_table) = workitem_container->if_swf_cnt_conversion~to_abap_container( ).

        LOOP AT container_table REFERENCE INTO DATA(container_ref) WHERE name = zwf_businesspartner_govern_c=>attach_objects.
          ASSIGN container_ref->value->* TO <attachment>.
          EXIT.
        ENDLOOP.

        IF <attachment> IS NOT ASSIGNED.
          RETURN.
        ENDIF.

        LOOP AT <attachment> REFERENCE INTO DATA(attachment_ref).
          DATA(gos_object) = VALUE borident( objtype = zwf_businesspartner_govern_c=>businessobjects-message
                                             objkey  = CONV #( attachment_ref->instid ) ).

          CALL FUNCTION 'BINARY_RELATION_CREATE_COMMIT'
            EXPORTING
              obj_rolea    = object
              obj_roleb    = gos_object
              relationtype = zwf_businesspartner_govern_c=>attachment_relation
            EXCEPTIONS
              OTHERS       = 1.

        ENDLOOP.

      CATCH cx_swf_run_wim INTO DATA(exception). " TODO: variable is assigned but never used (ABAP cleaner)
    ENDTRY.
