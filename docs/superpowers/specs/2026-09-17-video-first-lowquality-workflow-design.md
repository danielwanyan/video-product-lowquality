# Video-First Lowquality Workflow Design

Date: 2026-09-17

## Purpose

Optimize the `video-product-lowquality` Aicolate workflow for video low-quality review. The workflow should output one binary machine judgment per video, and human review will correct the result later. Human review JSON is the feedback loop for sharpening future rule boundaries.

This design intentionally narrows the workflow away from product quality or review-driven merchant risk. Product evidence is auxiliary only.

## Evidence From Current Evaluation

Source files used for this design:

- Input and Aicolate output Excel: `/Users/bytedance/Desktop/EU-LLM_Wiki/批量抓取电商数据Pearl+Holmes/Final_result_excel/Translated_merged_data/599videos_20260916_130200_with_review_zh_merged_aicolate_raw_output.xlsx`
- Human review JSON: `/Users/bytedance/Downloads/pearl_reviews_2026-09-17.json`
- Current GitHub rule repo: `/Users/bytedance/video-product-lowquality`
- Current wiki mirror and code-node references: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality`

Observed from `/Users/bytedance/Downloads/pearl_reviews_2026-09-17.json`:

| Item | Value |
|---|---:|
| Cases | 599 |
| Human clean | 438 |
| Human problematic | 161 |
| Aicolate clean | 181 |
| Aicolate problematic | 357 |
| Aicolate manual_review | 61 |
| Parsed outputs missing `human_review_labels` | 599 |
| Parsed outputs with invalid v12 primary issue type | 389 |
| Parsed outputs still containing old `issue_results` keys | 599 |

Observed from local rule files:

- `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json` is `2026-09-16-v12-binary-decision-calibration`.
- `/Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json` is still `2026-09-15-v11-human-review-calibration`.

Main conclusion: the live/batch workflow is still behaving like an older taxonomy and not like the v12 binary workflow.

## Scope

The workflow reviews video low-quality signals only.

In scope:

- Video-product mismatch / IPP-style mismatch.
- Misleading functionality or effect.
- Suspected pirated or reused content.
- Pure marketing pitch.
- No physical product display.
- Irrelevant product promotion.
- Unrealistic or continuity error.
- Sexual or vulgar hook.
- Disgusting or terrifying visual.
- Pirated content.
- Description not detailed.
- Out-of-app transaction.
- Still frame.

Out of scope:

- Product quality risk based on buyer reviews.
- Merchant-side quality or fulfillment risk.
- Product-review-summary-driven product labels.
- Comment-driven product risk.
- `manual_review` as a model output.

## Evidence Boundary

### Evidence Passed To The Final LLM

The final LLM may use only these fields as judgment evidence:

- `frame_list`
- `ASR`
- `OCR`
- `images`
- `country`

`frame_list`, `ASR`, and `OCR` are the primary video evidence.

`images` and `country` are auxiliary evidence:

- `images` helps compare the shown or described product with the bound product image.
- `images` helps check whether video claims conflict with visible product promises.
- `country` helps with language and market context, but must not be a standalone hit reason.

### Display Or Trace-Only Fields

These fields must not be passed to the final LLM for judgment:

- `url`
- `product_id`
- `seller_id_str`
- `comments`
- `product_review_summary`

They may remain available in the human review page or downstream trace artifacts. They are not model evidence.

`video_id` may be passed only as an identifier to copy into output JSON. It is not judgment evidence.

## Target Workflow

```text
Start
  ├─ HTTP: project_rules_json_fetch
  ├─ HTTP: project_rules_text_fetch
  ├─ Code: Video_Content_Builder
  ├─ Code: Product_Image_Aux_Builder
  └─ Code: Boundary_Attention_Builder
        ↓
Code: Evidence_Gate
        ↓
LLM: Unified_Video_Lowquality_Reviewer
        ↓
End
```

Remove or stop wiring these nodes into the final LLM path:

- `Comment_Review_Signal_Builder`
- old `Product_Evidence_Builder` if it accepts identifiers or review summary as risk evidence
- `Context_Builder`
- `Output_Validator`

The End node should link directly to the single output field from `Unified_Video_Lowquality_Reviewer`.

## Output Contract

The final LLM must return raw JSON only. It must not wrap JSON in markdown fences or add prose before or after the object.

Allowed `overall_decision` values:

```text
problematic
clean
```

`manual_review` is not allowed. If evidence is uncertain or weak, the model must choose `clean` and explain the boundary in `boundary_reason_cn`.

Allowed `primary_issue_type` values:

```text
video_product_mismatch
misleading_functionality_or_effect
suspected_pirated_or_reused_content
pure_marketing_pitch
no_physical_product_display
irrelevant_product_promotion
unrealistic_or_continuity_error
sexual_or_vulgar_hook
disgusting_or_terrifying_visual
pirated_content
description_not_detailed
out_of_app_transaction
still_frame
none
```

Target JSON shape:

```json
{
  "video_id": "",
  "overall_decision": "problematic|clean",
  "primary_issue_type": "",
  "detected_issue_types": [],
  "human_review_labels": {
    "ratings": [],
    "tagsContent": [],
    "tagsAttribute": []
  },
  "video_evidence_summary_cn": "",
  "product_aux_summary_cn": "",
  "boundary_reason_cn": "",
  "final_reason_cn": "",
  "data_quality": {
    "video_frames_available": true,
    "asr_available": true,
    "ocr_available": true,
    "product_images_available": true,
    "country_available": true
  }
}
```

The workflow does not output `tagsProduct`.

`human_review_labels.ratings` must be:

- `["video"]` when `tagsContent` is non-empty.
- `["ok"]` when `tagsContent` is empty.

`tagsAttribute` may contain `视频不可见` when video frames are missing or unusable. The model must not infer `图文AIGC`, `无问题AIGC`, or `非AIGC` unless a future upstream explicit attribute field is added.

## Human Review Label Mapping

| `primary_issue_type` | `tagsContent` |
|---|---|
| `video_product_mismatch` | `挂车商品与讲解商品不一致` |
| `misleading_functionality_or_effect` | `功能效果虚假夸大` |
| `suspected_pirated_or_reused_content` | `疑似盗版 / 疑似盗剪` |
| `pure_marketing_pitch` | `仅营销叫卖` |
| `no_physical_product_display` | `无实物展示（围绕商品讲解）` |
| `irrelevant_product_promotion` | `内容与商品无关推广` |
| `unrealistic_or_continuity_error` | `穿帮 / 不符现实` |
| `sexual_or_vulgar_hook` | `擦边低俗` |
| `disgusting_or_terrifying_visual` | `恶心恐怖` |
| `pirated_content` | `盗版内容` |
| `description_not_detailed` | `内容介绍不详细（低质）` |
| `out_of_app_transaction` | `站外引流交易` |
| `still_frame` | `静止帧` |
| `none` | no `tagsContent`; `ratings=["ok"]` |

`detected_issue_types` may contain multiple issue types when the evidence clearly supports more than one label.

## Boundary Attention Layer

`Boundary_Attention_Builder` should create a compact, sample-specific attention packet before the final LLM. It should not make final judgments. It should tell the model which low-quality categories are plausible, which boundaries are active, and what not to infer.

### IPP / Product Mismatch

Do not hit `video_product_mismatch` for:

- color-only differences;
- minor style, material, or texture differences;
- ordinary package-language differences;
- brand text differences unless the brand itself is the sold product/IP issue;
- a video showing more units than product images when the sold product can include those units;
- a different opening scene, prop, or outfit when it is not promoted as the sold product.

Hit only when the video clearly promotes or explains a main product that conflicts with the bound product images in core category, function, form, visible quantity, key specification, or promised use.

### Misleading Functionality Or Effect

Do not hit from product images alone. The issue must be supported by `frame_list`, `ASR`, or `OCR`.

Do not hit for:

- ordinary advertising tone;
- subjective praise;
- metaphor;
- fashion or fit claims without concrete false promise;
- expected cleaning, shaving, styling, cooking, or demonstration outcomes;
- minor numerical ambiguity that can be ordinary speech error.

Hit when video evidence includes a concrete and misleading claim about efficacy, speed, durability, medical/health effect, quantity, time, before/after result, safety, or physical capability.

### Suspected Pirated Or Reused Content

Do not hit for:

- replying to a user comment;
- showing another creator briefly then self-verifying;
- self-comparison or proof-style content;
- same creator in different scenes.

Hit when the video strongly suggests reused or pirated content, such as Chinese domestic studio recording cues, Chinese UI/environment not fitting the target market, blurry reposted footage, watermark-hiding layout, or obvious unrelated creator/content reuse.

### No Physical Product Display And Still Frame

AI-generated style does not equal no physical product display.

Do not hit `no_physical_product_display` if the video clearly shows the bound product or a substantive product-use scene.

Do not hit `still_frame` when sampled frames show camera movement, angle shift, lighting shift, pose shift, or product movement.

Hit `still_frame` only when the sampled frames are effectively static and do not provide substantive product demonstration.

### Pure Marketing Pitch

Hit only when nearly all ASR/OCR is price, urgency, stock, discount, order guidance, or sales pressure, with almost no product function, material, size, use scenario, operation method, or realistic product information.

Weak but real product information should usually prevent `pure_marketing_pitch`.

### Irrelevant Product Promotion

Hit when most of the video body is unrelated to the bound product and the product appears only briefly, at the end, or not substantively.

Do not hit if a later segment provides real product demonstration or explanation.

### Unrealistic Or Continuity Error

This label is lower priority than IPP, MFE, suspected pirated/reused content, no-product display, and pure marketing because the workflow only receives up to 8 sampled frames. Sparse frame sampling creates FP risk.

Hit only when all three conditions are met:

- strong visual evidence;
- the error affects consumer understanding of the product;
- the change cannot reasonably be explained by sampling interval, normal editing, camera angle, or missing intermediate action.

Do not hit for:

- ordinary action jumps between 8 sampled frames;
- missing intermediate actions;
- slight hand artifacts;
- lighting or angle changes;
- motion blur;
- process-video cuts that do not affect product understanding.

If uncertain, choose `clean`.

### Sexual/Vulgar, Disgusting/Terrifying, Out-Of-App

Do not hit `sexual_or_vulgar_hook` for ordinary try-on, gym, swimwear, shapewear, or body-fit display without clear vulgar/adult framing.

Do not hit `disgusting_or_terrifying_visual` for mild unattractive visuals. Require dominant strong discomfort, injury, dense-phobia-like visual, dirt, bodily, or horror-style content.

Do not hit `out_of_app_transaction` for TikTok in-app purchase guidance or private message requests. Require explicit external website, QR code, off-platform channel, or external transaction instruction.

## Evidence Gate

`Evidence_Gate` should only enforce evidence availability and forbidden inferences. It must not recommend `manual_review`.

Required behavior:

- If `frame_list` is empty or unusable, forbid visual claims and add `视频不可见` to the expected attribute context.
- If `ASR` and `OCR` are empty, forbid text-only rules such as pure marketing or out-of-app transaction from missing text.
- If `images` are missing, forbid product-image mismatch comparison.
- If evidence is incomplete and no concrete rule hit remains, instruct the LLM to choose `clean`.

## Node Responsibilities

### Video_Content_Builder

Inputs:

- `frame_list`
- `ASR`
- `OCR`

Outputs:

- `video_frame_urls`
- `video_frame_manifest`
- `video_text_panel`
- `video_signal_flags_json`
- `video_data_quality_json`

Responsibilities:

- parse and clean up to 8 frame URLs;
- describe frame ordering as sampled frames, not full video;
- extract ASR/OCR signal flags for low-quality categories;
- never make final risk decisions.

### Product_Image_Aux_Builder

Inputs:

- `images`
- `country`

Outputs:

- `product_image_urls`
- `product_image_manifest`
- `product_aux_panel`
- `product_aux_data_quality_json`

Responsibilities:

- parse product image URLs;
- summarize country context;
- prepare product-image comparison context for IPP and MFE;
- not use product ID, seller ID, comments, or product review summary.

### Boundary_Attention_Builder

Inputs:

- `video_text_panel`
- `video_signal_flags_json`
- `product_aux_panel`
- `product_aux_data_quality_json`
- `rules_json_body`
- `rules_text_body`

Outputs:

- `boundary_attention_packet`
- `active_boundary_groups`
- `must_not_infer`
- `positive_hit_tests`
- `allowed_issue_types`

Responsibilities:

- select relevant boundary groups;
- compress rule reminders into direct positive and negative tests;
- make boundary constraints visible to the final LLM;
- never use comments or product review summary.

### Evidence_Gate

Inputs:

- `video_frame_urls`
- `video_data_quality_json`
- `product_image_urls`
- `product_aux_data_quality_json`
- `boundary_attention_packet`

Outputs:

- `gated_all_image_urls`
- `gated_all_image_manifest`
- `evidence_gate_panel`
- `forbidden_claims`
- `allowed_issue_types`
- `final_reviewer_context`

Responsibilities:

- build the final image array from product images and video frames;
- block evidence claims for missing channels;
- combine boundary attention with evidence availability;
- instruct clean-by-default when strong evidence is missing.

### Unified_Video_Lowquality_Reviewer

Inputs:

- `gated_all_image_urls`
- `gated_all_image_manifest`
- `final_reviewer_context`
- `frame_list`
- `ASR`
- `OCR`
- `images`
- `country`
- `video_id` as identifier only

Outputs:

- one raw JSON field connected directly to End.

Responsibilities:

- inspect the visual inputs directly;
- produce binary judgment only;
- output the new issue taxonomy and human review labels;
- use Chinese evidence summaries and reasons;
- avoid all display-only fields as evidence.

## Manual Aicolate Update Strategy

After implementation, update Aicolate one node at a time.

For each node, provide:

- full replacement code or prompt;
- exact Input Variables;
- exact Output panel fields;
- wiring instructions;
- a stop point for confirmation before the next node.

Recommended update order:

1. `Video_Content_Builder`
2. `Product_Image_Aux_Builder`
3. `Boundary_Attention_Builder`
4. `Evidence_Gate`
5. `Unified_Video_Lowquality_Reviewer` System Prompt
6. `Unified_Video_Lowquality_Reviewer` User Prompt
7. End node wiring

Do not update `Output_Validator`; it should be removed from the active path.

## Verification Plan

Use local unit tests for code-node behavior before providing Aicolate copy-paste content.

Minimum checks:

- code nodes do not read `comments` or `product_review_summary`;
- code nodes do not treat `url`, `product_id`, or `seller_id_str` as LLM evidence;
- output schema contains only `problematic` and `clean`;
- `manual_review` is absent from prompts and rules for final output;
- `tagsProduct` is absent;
- boundary tests cover the repeated human-review corrections from the 599-case review set:
  - color-only difference is clean;
  - clear product display is not description-not-detailed;
  - AI style is not no-product-display;
  - reply/self-verification is not suspected piracy;
  - product image alone does not prove MFE;
  - pure marketing requires near-total sales pitch;
  - continuity errors require strong evidence and lower priority.

## Open Decisions

No open decision remains before implementation planning.

The user has approved:

- no `manual_review`;
- no `Output_Validator` node;
- comments and product review summary are display-only;
- URL, product ID, and seller ID are display-only;
- final LLM evidence is limited to `frame_list`, `ASR`, `OCR`, `images`, and `country`;
- manual Aicolate update instructions must be one node at a time with full copy-paste content.
