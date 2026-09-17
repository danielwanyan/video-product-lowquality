# Aicolate Manual Update Guide - Video Product Lowquality v13

## Step 0: Do Not Change These Display Fields

Keep `url`, `product_id`, `seller_id_str`, `comments`, and `product_review_summary` available only for human display or tracing. Do not wire them into the final LLM prompt.

The final LLM judgment evidence is limited to:

- `frame_list`
- `ASR`
- `OCR`
- `images`
- `country`

## Step 1: HTTP Rule Nodes

Use the same raw URLs:

- `https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_structured_v1.json`
- `https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_text_v1.txt`

After the repo is pushed, rerun the HTTP nodes and confirm the structured rule version is:

```text
2026-09-17-v13-video-first-binary-boundary-calibration
```

## Step 2: Video_Content_Builder

Input Variables:

- `frame_list = {{frame_list}}`
- `ASR = {{ASR}}`
- `OCR = {{OCR}}`
- `video_id = {{video_id}}`

Output panel:

- `video_frame_urls`
- `video_frame_manifest`
- `video_text_panel`
- `video_signal_flags_json`
- `video_data_quality_json`

Paste full code from:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/video_content_builder.py
```

Stop here and run one sample before continuing.

## Step 3: Product_Image_Aux_Builder

Input Variables:

- `images = {{images}}`
- `country = {{country}}`

Output panel:

- `product_image_urls`
- `product_image_manifest`
- `product_aux_panel`
- `product_aux_data_quality_json`

Paste full code from:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/product_image_aux_builder.py
```

Stop here and run one sample before continuing.

## Step 4: Boundary_Attention_Builder

Input Variables:

- `video_text_panel = {{video_text_panel}}`
- `video_data_quality_json = {{video_data_quality_json}}`
- `product_aux_panel = {{product_aux_panel}}`
- `product_aux_data_quality_json = {{product_aux_data_quality_json}}`
- `rules_json_body = {{project_rules_json_fetch.body}}`
- `rules_text_body = {{project_rules_text_fetch.body}}`

Output panel:

- `boundary_attention_packet`
- `active_boundary_groups`
- `must_not_infer`
- `positive_hit_tests`
- `allowed_issue_types`

Paste full code from:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/boundary_attention_builder.py
```

Stop here and run one sample before continuing.

## Step 5: Evidence_Gate

Input Variables:

- `video_frame_urls = {{video_frame_urls}}`
- `video_frame_manifest = {{video_frame_manifest}}`
- `video_data_quality_json = {{video_data_quality_json}}`
- `product_image_urls = {{product_image_urls}}`
- `product_image_manifest = {{product_image_manifest}}`
- `product_aux_data_quality_json = {{product_aux_data_quality_json}}`
- `boundary_attention_packet = {{boundary_attention_packet}}`
- `allowed_issue_types = {{allowed_issue_types}}`

Output panel:

- `gated_all_image_urls`
- `gated_all_image_manifest`
- `evidence_gate_panel`
- `forbidden_claims`
- `allowed_issue_types`
- `final_reviewer_context`

Paste full code from:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/evidence_gate.py
```

Stop here and run one sample before continuing.

## Step 6: Unified_Video_Lowquality_Reviewer

System Prompt:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_system_prompt.txt
```

User Prompt:

```text
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_user_prompt.txt
```

Input Variables:

- `gated_all_image_urls = {{gated_all_image_urls}}`
- `gated_all_image_manifest = {{gated_all_image_manifest}}`
- `final_reviewer_context = {{final_reviewer_context}}`
- `evidence_gate_panel = {{evidence_gate_panel}}`
- `allowed_issue_types = {{allowed_issue_types}}`
- `forbidden_claims = {{forbidden_claims}}`
- `video_id = {{video_id}}`
- `ASR = {{ASR}}`
- `OCR = {{OCR}}`
- `country = {{country}}`

Do not wire `url`, `product_id`, `seller_id_str`, `comments`, or `product_review_summary`.

## Step 7: End Node

Connect End directly to the single output field from `Unified_Video_Lowquality_Reviewer`.

Remove `Output_Validator` from the active path.

## Step 8: Smoke Test

Run three samples:

- a known clean color-only mismatch false positive;
- a known problematic MFE sample;
- a known frame-missing sample if available.

Expected:

- no `manual_review`;
- no `tagsProduct`;
- `ratings` is `["video"]` for issues and `["ok"]` for clean;
- display-only fields are not referenced in `final_reason_cn`.
