
# Translation Pipeline Flowchart

```mermaid
graph TD
    %% Start and Input
    A[Start: Input Received<br>Main_document_ref1_ref2_translation_direction_session_id_filename_user_id_conn] -->|translate_background_docx.py| B[Create_Session_Directory<br>os_path_join_DATA_DIR_session_id]

    %% File Uploads
    B --> C[Upload_Main_Document<br>upload_to_do_main_content_session_id_filename]
    C -->|DigitalOcean_Spaces<br>app.py_get_upload_url| D[Get_main_file_url]
    D --> E{Check_Reference_Documents}
    E -->|ref1_filename_exists| F[Upload_Ref1<br>upload_to_do_ref1_content_session_id_ref1_filename]
    E -->|ref2_filename_exists| G[Upload_Ref2<br>upload_to_do_ref2_content_session_id_ref2_filename]
    F -->|DigitalOcean_Spaces<br>app.py_get_upload_url| H[Get_ref1_file_url_ref1_file_size]
    G -->|DigitalOcean_Spaces<br>app.py_get_upload_url| I[Get_ref2_file_url_ref2_file_size]
  
    E--> |No_refs| NR[set_ref1_ref2=None]
    NR--> K[Determine_Translation_Languages<br>translation_source_translation_target]
    H --> J[Set ref1_ref2 to the file url]
    I --> J

    %% Database Logging
    J --> K[Determine_Translation_Languages<br>translation_source_translation_target]
    K --> L[Log_to_Database<br>INSERT_INTO_translations_session_id_started_at_user_id_etc]
    L -->|PostgreSQL| M[Commit_Transaction]

    %% Reference Processing
    M --> N{Check_Reference_Documents}
    N -->|Both_ref1_text_and_ref2_text| O[Process_Reference_Files                              ]
    O --> P{Ref1_is_PDF}
    P -->|Yes| Q[Extract_ref1_text<br>get_pdf_ocr_text_ref1_content <br> Server ://ocr-pdf-458036510645.asia-southeast1]
    P -->|No| R[Extract_ref1_text<br>md_convert_BytesIO_ref1_content]
    Q --> T{Ref2 is pdf}
    R --> T
    
    T -->|Yes| U[Extract_ref2_text<br>get_pdf_ocr_text_ref2_content <br> Server ://ocr-pdf-458036510645.asia-southeast1]
    T -->|No| V[Extract_ref2_text<br>md_convert_BytesIO_ref2_content]
    U --> X[Log_Reference_Text_Lengths]
    V --> X[Log_Reference_Text_Lengths]
    X --> Y[Call_Reference_Pairs_API<br> ://ocr-pdf-7j4l.onrender.com/reference_pairs_api/call_llm_to_create_reference_pairs_streaming]
    Y -->|app.py_process_ocr| Z[Stream_Response_to_reference-beat_json<br>Extract_llm_pair_job_id_pair_string]
    Z -->|Error| AA[Write_Error_to_error_json]
    Z --> AB{Check_API_Response}
    AB -->|Success| AC[Parse_reference-beat_json_for_errors]
    AC -->|Error_found| AA
    N -->|No_refs| AD[Set_pair_string_to_empty]

    %% Translation Process
    AC --> AE
    AA --> AE
    AD --> AE[Determine_Source_Target_Languages<br>English_or_Chinese]
    AE --> AF[Call_Translation_API<br> ://ocr-pdf-7j4l.onrender.com/translation_api/translate_streaming]
    AF -->|translation_api.py_translate_document_streaming| AG[Download_Document<br>requests_get_original_docx_url]
    AG --> AH[Save_to_temp_downloads_filename_timestamp_docx]
    AH --> AI[Call_translate_document_async ]
    AI --> AJ[Read_DOCX<br>Document_original_docx_path]
    AJ --> AK[Collect_Text<br>collect_all_docx_texts_doc]
    AK --> AL[Group_Paragraphs<br>group_paragraphs_by_word_count<br>Max_1000_words_or_40_paragraphs]
    AL --> AM[Send_Batches_to_FastAPI<br>POST <br>API_ENDPOINT = ://170.187.224.24:23477/api/translate<br> PROGRESS_ENDPOINT = ://170.187.224.24:23477/api/jobs ]
    AM -->|fastapi_server_translation_0522.py_translate_batch| AN[Generate_job_id<br>Initialize_active_translations]
    AN --> AO[Process_Batches_in_Parallel<br>process_with_semaphore]
    AO --> AP[Call_Wang_Yi_Code_API<br>POST <br> ://146.190.101.128:1880/v1/chat/glossary_agent ]
    AP --> AQ[Construct_LLM_Prompt<br>system_prompt_user_input_llm_pair_result]
    AQ --> AR[Call_LLM<br>AsyncOpenAI_chat_completions_create]
    AR --> AS[Validate_Translation_Quality<br>validate_translation_quality]
    AS -->|Invalid| AT[Retry_up_to_3_times]
    AT -->|Max_retries_reached| AU[Use_last_translation]
    AS -->|Valid| AU
    AU --> AV[Parse_Results<br>parse_translation_result]
    AV --> AW[Save_Results<br>translation_results_job_id_json]
    AW --> AX[Update_Progress<br>progress_tracker_update_progress]
    AX --> AY[Check_Job_Status<br>GET_api_jobs_job_id_status]
    AY -->|Running| AZ[Wait_and_Check_Again]
    AY -->|Completed| BA[Get_Results<br>GET_api_jobs_job_id_results]
    BA --> BB[Apply_Translations<br>update_text]
    BB --> BC[Save_Translated_DOCX<br>doc_save_translated_docx_path]
    BC --> BD[Upload_to_Spaces<br>upload_file_to_spaces]
    BD --> BE[Get_download_url_file_key]
    BE -->|translation_api.py| BF[Stream_Response<br>translate-beat_json]
    BF --> BG[Save_Final_Response<br>translate-response_json]
    BG --> BH[Update_Database<br>UPDATE_translations_completed_at_translated_file_url]
    BH --> BI[Return_download_url]
    BI --> BJ[End: Translation_Complete]

    %% Error Handling
    AF -->|Error| BK[Write_Error_to_error_json]
    BK --> BH
    AR -->|Error| BL[Retry_up_to_3_times]
    BL -->|Max_retries_reached| BM[Return_empty_translation]
    BM --> AW
