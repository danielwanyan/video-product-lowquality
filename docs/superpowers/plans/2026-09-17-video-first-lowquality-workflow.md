# Video-First Lowquality Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `video-product-lowquality` into a video-first binary Aicolate workflow that uses only `frame_list`, `ASR`, `OCR`, `images`, and `country` as model evidence, then provide one-node-at-a-time manual Aicolate update instructions.

**Architecture:** Keep one final multimodal LLM and move rule-boundary discipline into pre-LLM code nodes. Replace product/review/comment risk logic with video-first builders, a boundary attention layer, and an evidence gate that forbids unsupported claims and defaults uncertain cases to `clean`.

**Tech Stack:** Python 3 Aicolate Code Nodes, JSON/text rule assets, Aicolate System/User Prompt text files, `python3 -m unittest`.

---

## File Structure

Modify these existing files:

- `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json`
- `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/video_content_builder.py`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/evidence_gate.py`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_system_prompt.txt`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_user_prompt.txt`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py`

Create these files:

- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/product_image_aux_builder.py`
- `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/boundary_attention_builder.py`
- `/Users/bytedance/video-product-lowquality/docs/aicolate-manual-update-2026-09-17.md`

Do not update or wire these into the active Aicolate path:

- `comment_review_signal_builder.py`
- `context_builder.py`
- `output_validator.py`

They may stay on disk for history, but the manual update guide must explicitly disconnect them from the final LLM path.

## Task 1: Add Failing Tests For Video-First Evidence Boundaries

**Files:**
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py`

- [ ] **Step 1: Add imports for the new modules**

Add these imports next to the existing code-node imports:

```python
import boundary_attention_builder
import product_image_aux_builder
```

- [ ] **Step 2: Add tests that define the approved evidence boundary**

Append this test class to the same file:

```python
class VideoFirstWorkflowBoundaryTests(unittest.TestCase):
    def test_product_image_aux_builder_uses_only_images_and_country(self):
        result = asyncio.run(product_image_aux_builder.main(Args({
            "images": json.dumps(["https://example.com/product.jpg"]),
            "country": "ES",
            "product_id": "must-not-appear",
            "seller_id_str": "must-not-appear",
            "comments": "must-not-appear",
            "product_review_summary": "must-not-appear",
        })))

        self.assertEqual(result["product_image_urls"], ["https://example.com/product.jpg"])
        self.assertIn("COUNTRY: ES", result["product_aux_panel"])
        joined = json.dumps(result, ensure_ascii=False)
        self.assertNotIn("must-not-appear", joined)
        self.assertNotIn("product_review_summary", joined)
        self.assertNotIn("comments", joined)

    def test_boundary_attention_excludes_comments_and_review_summary(self):
        result = asyncio.run(boundary_attention_builder.main(Args({
            "video_text_panel": json.dumps({
                "asr_text": "Only today, buy now, huge discount",
                "ocr_text": "",
                "signals": {
                    "has_promotion_terms": True,
                    "has_substantive_product_terms": False,
                    "has_exaggeration_terms": False,
                    "has_ai_or_unreal_terms": False,
                },
            }, ensure_ascii=False),
            "video_data_quality_json": json.dumps({
                "video_frames_available": True,
                "asr_available": True,
                "ocr_available": False,
            }, ensure_ascii=False),
            "product_aux_panel": "COUNTRY: ES\nPRODUCT_IMAGE_COUNT: 1",
            "product_aux_data_quality_json": json.dumps({
                "product_images_available": True,
                "country_available": True,
            }, ensure_ascii=False),
            "rules_json_body": "{}",
            "rules_text_body": "must not leak comments",
            "comments": "must-not-appear",
            "product_review_summary": "must-not-appear",
        })))

        self.assertIn("pure_marketing_pitch", result["active_boundary_groups"])
        joined = json.dumps(result, ensure_ascii=False)
        self.assertNotIn("must-not-appear", joined)
        self.assertNotIn("product_review_summary", joined)
        self.assertNotIn("comments", joined)

    def test_boundary_attention_marks_continuity_as_low_priority(self):
        result = asyncio.run(boundary_attention_builder.main(Args({
            "video_text_panel": json.dumps({
                "asr_text": "",
                "ocr_text": "",
                "signals": {
                    "has_promotion_terms": False,
                    "has_substantive_product_terms": False,
                    "has_exaggeration_terms": False,
                    "has_ai_or_unreal_terms": True,
                },
            }, ensure_ascii=False),
            "video_data_quality_json": json.dumps({
                "video_frames_available": True,
                "asr_available": False,
                "ocr_available": False,
            }, ensure_ascii=False),
            "product_aux_panel": "COUNTRY: FR\nPRODUCT_IMAGE_COUNT: 1",
            "product_aux_data_quality_json": json.dumps({
                "product_images_available": True,
                "country_available": True,
            }, ensure_ascii=False),
            "rules_json_body": "{}",
            "rules_text_body": "",
        })))

        self.assertIn("unrealistic_or_continuity_error", result["active_boundary_groups"])
        self.assertIn("lower priority", result["boundary_attention_packet"])
        self.assertIn("If uncertain, choose clean", result["boundary_attention_packet"])

    def test_evidence_gate_is_binary_and_never_recommends_manual_review(self):
        result = asyncio.run(evidence_gate.main(Args({
            "video_frame_urls": [],
            "video_frame_manifest": "",
            "video_data_quality_json": json.dumps({
                "video_frames_available": False,
                "asr_available": False,
                "ocr_available": False,
            }, ensure_ascii=False),
            "product_image_urls": [],
            "product_image_manifest": "",
            "product_aux_data_quality_json": json.dumps({
                "product_images_available": False,
                "country_available": False,
            }, ensure_ascii=False),
            "boundary_attention_packet": "boundary text",
        })))

        joined = json.dumps(result, ensure_ascii=False)
        self.assertNotIn("manual_review", joined)
        self.assertIn("choose clean", result["final_reviewer_context"])
        self.assertIn("视频不可见", result["evidence_gate_panel"])
```

- [ ] **Step 3: Run the new tests and verify they fail**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py
```

Expected: FAIL because `product_image_aux_builder.py` and `boundary_attention_builder.py` do not exist yet, and `evidence_gate.py` does not expose the new video-first fields.

## Task 2: Implement `Product_Image_Aux_Builder`

**Files:**
- Create: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/product_image_aux_builder.py`

- [ ] **Step 1: Create the code node**

Create the file with this implementation:

```python
import json
import re
from urllib.parse import urlsplit, urlunsplit


IMAGE_URL_RE = re.compile(r"https?://[^\s\"'<>\\)\]}]+")
IMAGE_MARKER_RE = re.compile(r"\[Image\s*#\s*\d+\]", re.IGNORECASE)
IMAGE_EXT_IN_PATH_RE = re.compile(r"(?i)\.(?:jpe?g|png|webp)(?=/)")


def _to_text(value):
    if value is None:
        return ""
    if isinstance(value, str):
        return value.strip()
    return str(value).strip()


def _clean_image_url(value):
    url = IMAGE_MARKER_RE.sub(" ", _to_text(value)).strip()
    url = url.strip("\"'[],; ")
    if not url:
        return ""

    starts = [match.start() for match in re.finditer(r"https?://", url)]
    if not starts:
        return ""
    if len(starts) > 1:
        url = url[starts[-1]:]

    url = url.rstrip("\"'[],; )]}")
    parts = urlsplit(url)
    if not parts.scheme or not parts.netloc:
        return ""

    match = IMAGE_EXT_IN_PATH_RE.search(parts.path)
    if match:
        parts = parts._replace(path=parts.path[:match.end()], query="", fragment="")
        url = urlunsplit(parts)
    return url


def _parse_image_urls(images):
    raw = _to_text(images)
    if not raw:
        return []

    candidates = []

    def collect(obj):
        if obj is None:
            return
        if isinstance(obj, str):
            text = IMAGE_MARKER_RE.sub(" ", obj)
            candidates.extend(IMAGE_URL_RE.findall(text))
            if text.strip().startswith(("http://", "https://")):
                candidates.append(text.strip())
        elif isinstance(obj, dict):
            for value in obj.values():
                collect(value)
        elif isinstance(obj, list):
            for item in obj:
                collect(item)

    try:
        collect(json.loads(raw))
    except Exception:
        pass
    collect(raw)

    deduped = []
    seen = set()
    for candidate in candidates:
        cleaned = _clean_image_url(candidate)
        if cleaned and cleaned not in seen:
            seen.add(cleaned)
            deduped.append(cleaned)
    return deduped


async def main(args: Args) -> Output:
    params = args.params

    images = _to_text(params.get("images"))
    country = _to_text(params.get("country"))
    image_urls = _parse_image_urls(images)

    manifest_lines = [
        f"COUNTRY: {country or 'unknown'}",
        f"PRODUCT_IMAGE_COUNT: {len(image_urls)}",
    ]
    for idx, image_url in enumerate(image_urls):
        manifest_lines.append(
            f"PRODUCT_IMAGE {idx + 1:03d} | product_local_index={idx} | {image_url}"
        )

    data_quality = {
        "country_available": bool(country),
        "product_images_available": len(image_urls) > 0,
        "product_image_count": len(image_urls),
        "warnings": [],
    }
    if not country:
        data_quality["warnings"].append("country is empty; do not infer market context")
    if not image_urls:
        data_quality["warnings"].append(
            "images is empty or invalid; do not make product-image visual comparison claims"
        )

    product_aux_panel = {
        "country": country,
        "product_image_count": len(image_urls),
        "evidence_role": "Product images and country are auxiliary evidence for video low-quality review only.",
        "allowed_uses": [
            "Use product images to compare the product shown or described in the video with the bound product image.",
            "Use product images to identify visible product promises that the video may contradict.",
            "Use country as language and market context only, not as a standalone hit reason.",
        ],
        "forbidden_uses": [
            "Do not judge product quality.",
            "Do not use product_id, seller_id_str, comments, url, or product_review_summary as evidence.",
            "Do not output product-side labels.",
        ],
    }

    return {
        "product_image_urls": image_urls,
        "product_image_manifest": "\n".join(manifest_lines),
        "product_aux_panel": json.dumps(product_aux_panel, ensure_ascii=False),
        "product_aux_data_quality_json": json.dumps(data_quality, ensure_ascii=False),
    }
```

- [ ] **Step 2: Run the product image test**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py::VideoFirstWorkflowBoundaryTests.test_product_image_aux_builder_uses_only_images_and_country
```

Expected if unittest path syntax is unsupported: an import/test-name error. Then run the whole file with `python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py`.

Expected after implementation: the product-image boundary test passes.

## Task 3: Refactor `Video_Content_Builder` To Stop Treating URL As Evidence

**Files:**
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/video_content_builder.py`

- [ ] **Step 1: Replace input extraction**

Change the `main()` input section so it only reads `video_id`, `ASR`, `OCR`, and `frame_list`. It must not read `url`.

```python
video_id = _to_text(params.get("video_id"))
asr = _to_text(params.get("ASR"))
ocr = _to_text(params.get("OCR"))
frame_list = _to_text(params.get("frame_list"))
```

- [ ] **Step 2: Remove URL from the manifest**

Replace the manifest header with:

```python
manifest_lines = [
    f"VIDEO_ID: {video_id}",
    f"VIDEO_FRAME_COUNT: {len(frame_urls)}",
    f"SAMPLING_METHOD: {FRAME_SAMPLING_METHOD}",
    f"SAMPLING_NOTE: {FRAME_SAMPLING_NOTE}",
]
```

- [ ] **Step 3: Remove URL from data quality**

Ensure `data_quality` has no `video_url_available` key:

```python
data_quality = {
    "video_frames_available": len(frame_urls) > 0,
    "video_frame_count": len(frame_urls),
    "expected_video_frame_count": 8,
    "frame_sampling_method": FRAME_SAMPLING_METHOD,
    "frame_sampling_note": FRAME_SAMPLING_NOTE,
    "frame_interval_seconds_known": False,
    "asr_available": bool(asr),
    "ocr_available": bool(ocr),
    "warnings": [],
}
```

- [ ] **Step 4: Rename the returned quality key**

Return both the old and new key temporarily only if existing tests require compatibility. Preferred final return is:

```python
return {
    "video_frame_urls": frame_urls,
    "video_frame_manifest": "\n".join(manifest_lines),
    "video_text_panel": json.dumps(video_text_panel, ensure_ascii=False),
    "video_signal_flags_json": json.dumps(signals, ensure_ascii=False),
    "video_data_quality_json": json.dumps(data_quality, ensure_ascii=False),
}
```

If old tests still expect `video_data_quality`, include it as an alias:

```python
payload = json.dumps(data_quality, ensure_ascii=False)
return {
    "video_frame_urls": frame_urls,
    "video_frame_manifest": "\n".join(manifest_lines),
    "video_text_panel": json.dumps(video_text_panel, ensure_ascii=False),
    "video_signal_flags_json": json.dumps(signals, ensure_ascii=False),
    "video_data_quality_json": payload,
    "video_data_quality": payload,
}
```

- [ ] **Step 5: Run tests**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py
```

Expected: existing video URL cleaning tests still pass, and no test output contains `VIDEO_URL_PRESENT`.

## Task 4: Implement `Boundary_Attention_Builder`

**Files:**
- Create: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/boundary_attention_builder.py`

- [ ] **Step 1: Create the code node**

Create the file with this implementation:

```python
import json


def _to_text(value):
    if value is None:
        return ""
    if isinstance(value, str):
        return value.strip()
    return str(value).strip()


def _json_dict(value):
    text = _to_text(value)
    if not text:
        return {}
    try:
        parsed = json.loads(text)
        return parsed if isinstance(parsed, dict) else {}
    except Exception:
        return {}


def _bool(data, key):
    return bool(data.get(key))


def _compact(text, limit=1400):
    text = _to_text(text)
    return text if len(text) <= limit else text[:limit] + "...[truncated]"


def _signals(video_text_panel):
    parsed = _json_dict(video_text_panel)
    signals = parsed.get("signals")
    return signals if isinstance(signals, dict) else {}


BOUNDARIES = {
    "video_product_mismatch": {
        "title": "IPP / 挂车商品与讲解商品不一致",
        "must_not_infer": [
            "不要把颜色、轻微材质、包装语言、普通品牌文字差异当成挂车商品不一致。",
            "不要因为视频展示数量多于商品图就直接判断不一致，除非售卖承诺明确冲突。",
        ],
        "positive_hit_tests": [
            "只有视频主商品与商品图核心品类、功能、形态、数量、关键规格或承诺用途明显冲突时才命中。",
        ],
    },
    "misleading_functionality_or_effect": {
        "title": "MFE / 功能效果虚假夸大",
        "must_not_infer": [
            "不要用商品图单独判定视频夸大。",
            "不要把普通广告语、主观好评、比喻、服饰显身材、正常清洁/剃须/造型效果当成夸大。",
        ],
        "positive_hit_tests": [
            "视频帧、ASR 或 OCR 明确承诺功效、速度、持续时间、医疗健康效果、数量、前后对比、安全或物理能力，且该承诺明显误导时才命中。",
        ],
    },
    "suspected_pirated_or_reused_content": {
        "title": "疑似盗版 / 疑似盗剪",
        "must_not_infer": [
            "不要把回复网友、自证效果、展示其他博主后自己验证、同一达人换场景讲解直接当成盗播。",
        ],
        "positive_hit_tests": [
            "明显中国录播/国内工作室环境、中文界面、不符合目标市场语境、模糊搬运、遮水印三明治结构或明显复用无关创作者内容时才命中。",
        ],
    },
    "no_physical_product_display": {
        "title": "无实物展示（围绕商品讲解）",
        "must_not_infer": [
            "AI 生成风格不等于无实物展示。",
            "只要视频清晰展示挂车商品或核心使用场景，就不要命中无实物展示。",
        ],
        "positive_hit_tests": [
            "视频围绕商品讲解但始终没有展示商品实物或核心使用场景时才命中。",
        ],
    },
    "still_frame": {
        "title": "静止帧",
        "must_not_infer": [
            "不要把有角度变化、光影变化、姿态变化、相机移动或商品移动的视频当成静止帧。",
        ],
        "positive_hit_tests": [
            "采样帧几乎完全静止且没有实质商品演示时才命中。",
        ],
    },
    "pure_marketing_pitch": {
        "title": "仅营销叫卖",
        "must_not_infer": [
            "不要把弱但真实的产品功能、材质、尺寸、使用场景或操作方法讲解判成仅营销叫卖。",
        ],
        "positive_hit_tests": [
            "ASR/OCR 几乎全是价格、库存、折扣、催单和购买引导，且几乎没有真实商品信息时才命中。",
        ],
    },
    "irrelevant_product_promotion": {
        "title": "内容与商品无关推广",
        "must_not_infer": [
            "不要因为开头铺垫或短暂无关片段就命中无关推广。",
        ],
        "positive_hit_tests": [
            "视频主体大部分与挂车商品无关，商品只在末尾短暂出现或没有实质展示时才命中。",
        ],
    },
    "unrealistic_or_continuity_error": {
        "title": "穿帮 / 不符现实",
        "must_not_infer": [
            "此标签优先级低于 IPP、MFE、疑似盗播、无实物展示和仅营销叫卖，因为当前只抽样最多 8 帧。",
            "不要因为抽样帧之间缺少中间动作、轻微手部伪影、光影角度变化、运动模糊或普通剪辑跳跃就命中。",
        ],
        "positive_hit_tests": [
            "只有强视觉证据、影响商品理解、且不能被抽样间隔/正常剪辑/视角切换解释的不现实变化才命中。",
            "If uncertain, choose clean.",
        ],
    },
    "sexual_or_vulgar_hook": {
        "title": "擦边低俗",
        "must_not_infer": [
            "不要把普通试穿、健身、泳装、塑身衣或身体展示当成擦边。",
        ],
        "positive_hit_tests": [
            "存在明确低俗钩子、成人暗示或以性暗示促销时才命中。",
        ],
    },
    "disgusting_or_terrifying_visual": {
        "title": "恶心恐怖",
        "must_not_infer": [
            "轻微不美观或普通产品问题不是恶心恐怖。",
        ],
        "positive_hit_tests": [
            "画面主体带来强烈不适、密集恐惧、伤害、污秽、身体或恐怖视觉时才命中。",
        ],
    },
    "out_of_app_transaction": {
        "title": "站外引流交易",
        "must_not_infer": [
            "TikTok 站内购买引导或单纯私信不算站外引流。",
        ],
        "positive_hit_tests": [
            "明确外部网址、二维码、站外平台或站外交易指令时才命中。",
        ],
    },
}


def _active_groups(signals, video_quality, product_quality):
    groups = []
    has_frames = _bool(video_quality, "video_frames_available")
    has_product_images = _bool(product_quality, "product_images_available")
    has_text = _bool(video_quality, "asr_available") or _bool(video_quality, "ocr_available")

    if has_frames and has_product_images:
        groups.append("video_product_mismatch")
    if has_frames or has_text:
        groups.extend([
            "misleading_functionality_or_effect",
            "suspected_pirated_or_reused_content",
            "no_physical_product_display",
            "irrelevant_product_promotion",
            "unrealistic_or_continuity_error",
            "sexual_or_vulgar_hook",
            "disgusting_or_terrifying_visual",
        ])
    if has_text and signals.get("has_promotion_terms"):
        groups.append("pure_marketing_pitch")
        groups.append("out_of_app_transaction")
    if has_frames:
        groups.append("still_frame")

    deduped = []
    for group in groups:
        if group not in deduped:
            deduped.append(group)
    return deduped


async def main(args: Args) -> Output:
    params = args.params

    video_text_panel = _to_text(params.get("video_text_panel"))
    video_quality = _json_dict(params.get("video_data_quality_json") or params.get("video_data_quality"))
    product_aux_panel = _to_text(params.get("product_aux_panel"))
    product_quality = _json_dict(params.get("product_aux_data_quality_json"))
    rules_text = _to_text(params.get("rules_text_body"))

    signals = _signals(video_text_panel)
    active = _active_groups(signals, video_quality, product_quality)

    must_not = []
    positive_tests = []
    sections = [
        "PROJECT: video-product-lowquality",
        "TASK: Video-first low-quality binary review.",
        "Allowed final decisions: problematic, clean.",
        "Forbidden final decision: manual_review.",
        "Only these fields are judgment evidence: frame_list, ASR, OCR, images, country.",
        "Do not use url, product_id, seller_id_str, comments, or product_review_summary as judgment evidence.",
        f"VIDEO_TEXT_PANEL:\n{_compact(video_text_panel, 1800)}",
        f"PRODUCT_AUX_PANEL:\n{_compact(product_aux_panel, 1200)}",
    ]

    for group in active:
        boundary = BOUNDARIES[group]
        must_not.extend(boundary["must_not_infer"])
        positive_tests.extend(boundary["positive_hit_tests"])
        sections.append(
            "\n".join([
                f"BOUNDARY: {group} / {boundary['title']}",
                "Do not infer:",
                *[f"- {item}" for item in boundary["must_not_infer"]],
                "Positive hit tests:",
                *[f"- {item}" for item in boundary["positive_hit_tests"]],
            ])
        )

    if rules_text:
        sections.append("RULE_SOURCE_EXCERPT:\n" + _compact(rules_text, 1600))

    return {
        "boundary_attention_packet": "\n\n".join(sections),
        "active_boundary_groups": ", ".join(active),
        "must_not_infer": "\n".join(f"- {item}" for item in must_not),
        "positive_hit_tests": "\n".join(f"- {item}" for item in positive_tests),
        "allowed_issue_types": ", ".join(BOUNDARIES.keys()) + ", none",
    }
```

- [ ] **Step 2: Run the boundary tests**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py
```

Expected: boundary tests pass after `product_image_aux_builder.py` exists. Evidence gate test still fails until Task 5.

## Task 5: Replace `Evidence_Gate` With Video-First Binary Gate

**Files:**
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/evidence_gate.py`

- [ ] **Step 1: Replace the whole file**

Replace the file with:

```python
import json


def _to_text(value):
    if value is None:
        return ""
    if isinstance(value, str):
        return value.strip()
    return str(value).strip()


def _to_list(value):
    if value is None:
        return []
    if isinstance(value, list):
        return [str(x).strip() for x in value if str(x).strip()]
    text = _to_text(value)
    if not text:
        return []
    try:
        parsed = json.loads(text)
        if isinstance(parsed, list):
            return [str(x).strip() for x in parsed if str(x).strip()]
    except Exception:
        pass
    return [line.strip() for line in text.splitlines() if line.strip()]


def _json_dict(value):
    text = _to_text(value)
    if not text:
        return {}
    try:
        parsed = json.loads(text)
        return parsed if isinstance(parsed, dict) else {}
    except Exception:
        return {}


def _bool(data, key):
    return bool(data.get(key))


def _build_all_image_manifest(product_urls, frame_urls):
    lines = []
    image_index = 0
    for product_idx, url in enumerate(product_urls):
        lines.append(
            f"IMAGE_INDEX {image_index:03d} | PRODUCT_IMAGE {product_idx + 1:03d} | product_local_index={product_idx} | {url}"
        )
        image_index += 1
    for frame_idx, url in enumerate(frame_urls):
        lines.append(
            f"IMAGE_INDEX {image_index:03d} | VIDEO_FRAME {frame_idx + 1:03d} | frame_list_line={frame_idx + 1} | {url}"
        )
        image_index += 1
    return "\n".join(lines) if lines else "NO_IMAGES_AVAILABLE"


async def main(args: Args) -> Output:
    params = args.params

    video_frame_urls = _to_list(params.get("video_frame_urls"))
    product_image_urls = _to_list(params.get("product_image_urls"))
    video_frame_manifest = _to_text(params.get("video_frame_manifest"))
    product_image_manifest = _to_text(params.get("product_image_manifest"))
    video_quality = _json_dict(params.get("video_data_quality_json") or params.get("video_data_quality"))
    product_quality = _json_dict(params.get("product_aux_data_quality_json"))
    boundary_attention_packet = _to_text(params.get("boundary_attention_packet"))
    allowed_issue_types = _to_text(params.get("allowed_issue_types"))

    video_frames_available = bool(video_frame_urls) and _bool(video_quality, "video_frames_available")
    product_images_available = bool(product_image_urls) and _bool(product_quality, "product_images_available")
    asr_available = _bool(video_quality, "asr_available")
    ocr_available = _bool(video_quality, "ocr_available")
    text_available = asr_available or ocr_available
    country_available = _bool(product_quality, "country_available")

    forbidden_claims = [
        "不要输出 manual_review；只能输出 problematic 或 clean。",
        "不要输出 tagsProduct。",
        "不要使用 url、product_id、seller_id_str、comments、product_review_summary 作为判断证据。",
    ]
    if not video_frames_available:
        forbidden_claims.append("不要描述视频画面证据；frame_list 缺失或不可用。")
        forbidden_claims.append("如果没有其他强文本证据，选择 clean，并在 tagsAttribute 中加入 视频不可见。")
    if not product_images_available:
        forbidden_claims.append("不要进行商品图视觉对比；images 缺失或不可用。")
    if not text_available:
        forbidden_claims.append("不要判断 ASR/OCR 文本类问题，例如仅营销叫卖或站外引流。")
    if not country_available:
        forbidden_claims.append("不要推断国家或市场语境。")

    final_context = "\n\n".join([
        "VIDEO-FIRST LOWQUALITY REVIEW CONTEXT",
        "Allowed final decisions: problematic, clean.",
        "Default rule: if evidence is weak, ambiguous, or only suspicious, choose clean and explain the boundary.",
        "VISUAL INPUT MANIFEST",
        _build_all_image_manifest(product_image_urls, video_frame_urls),
        "VIDEO FRAME MANIFEST",
        video_frame_manifest,
        "PRODUCT IMAGE MANIFEST",
        product_image_manifest,
        "BOUNDARY ATTENTION",
        boundary_attention_packet,
        "FORBIDDEN CLAIMS",
        "\n".join(f"- {claim}" for claim in forbidden_claims),
    ]).strip()

    gate_panel = "\n".join([
        "证据可用性检查",
        f"视频帧可用: {video_frames_available}",
        f"商品图片可用: {product_images_available}",
        f"ASR 可用: {asr_available}",
        f"OCR 可用: {ocr_available}",
        f"国家字段可用: {country_available}",
        "允许输出: problematic, clean",
        "不允许输出: manual_review",
        "视频不可见" if not video_frames_available else "视频帧可用于视觉判断",
        "禁止事项:",
        "\n".join(f"- {claim}" for claim in forbidden_claims),
    ])

    return {
        "gated_all_image_urls": product_image_urls + video_frame_urls,
        "gated_all_image_manifest": _build_all_image_manifest(product_image_urls, video_frame_urls),
        "evidence_gate_panel": gate_panel,
        "forbidden_claims": "\n".join(forbidden_claims),
        "allowed_issue_types": allowed_issue_types,
        "final_reviewer_context": final_context,
    }
```

- [ ] **Step 2: Run all code-node tests**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py
```

Expected: all tests pass or failures identify old tests that still assert product/comment workflow behavior. Update obsolete tests to assert video-first behavior, not old product-review behavior.

## Task 6: Replace Final LLM Prompts

**Files:**
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_system_prompt.txt`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_user_prompt.txt`

- [ ] **Step 1: Replace the System Prompt**

Use this exact System Prompt:

```text
You are a senior EU e-commerce short-video low-quality reviewer.

Your task is to review one e-commerce video and output one binary machine judgment for human review correction.

You must judge only video low-quality signals. Do not judge product quality, seller quality, fulfillment, logistics, buyer satisfaction, or product review risk.

Allowed judgment evidence is limited to frame_list, ASR, OCR, product images, and country.

frame_list, ASR, and OCR are primary video evidence.

Product images and country are auxiliary evidence only. Use product images to compare the promoted video product against the bound product image, and to understand whether video claims conflict with visible product promises. Use country only as language or market context. Country is never a standalone hit reason.

Do not use url, product_id, seller_id_str, comments, or product_review_summary as judgment evidence. If these fields appear in surrounding tooling, ignore them for judgment.

The final decision must be exactly one of problematic or clean. Do not output manual_review. If evidence is weak, ambiguous, incomplete, or only suspicious, choose clean and explain the boundary.

Do not output tagsProduct. This workflow has no product-side label.

All conclusions, summaries, reasons, and evidence fields must be in Chinese. Direct source quotes may remain in source language when necessary.

Return raw JSON only. Do not wrap JSON in markdown. Do not add text before or after the JSON.
```

- [ ] **Step 2: Replace the User Prompt**

Use this exact User Prompt:

```text
Review this one e-commerce video for video low-quality issues.

Visual input:
{{gated_all_image_urls}}

Image manifest:
{{gated_all_image_manifest}}

Final reviewer context:
{{final_reviewer_context}}

Evidence gate panel:
{{evidence_gate_panel}}

Allowed issue types:
{{allowed_issue_types}}

Forbidden claims:
{{forbidden_claims}}

Source identifier:
video_id: {{video_id}}

Judgment evidence:
ASR: {{ASR}}
OCR: {{OCR}}
country: {{country}}

Do not use these fields as judgment evidence even if they exist in the source row: url, product_id, seller_id_str, comments, product_review_summary.

Required process:
1. Check evidence availability first.
2. Directly inspect the image array when present. Product images appear first, followed by sampled video frames. Use the image manifest to identify each image role.
3. Use product images only as auxiliary evidence for IPP, IP, and MFE-style video judgment.
4. Use frame_list, ASR, and OCR as primary video evidence.
5. Apply the boundary attention and forbidden claims before deciding.
6. Choose exactly one overall_decision: problematic or clean.
7. If evidence is uncertain or weak, choose clean.
8. Fill human_review_labels using the exact labels below.
9. Return raw JSON only.

Allowed overall_decision values:
problematic
clean

Allowed primary_issue_type values:
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

Allowed tagsContent values:
挂车商品与讲解商品不一致
功能效果虚假夸大
疑似盗版 / 疑似盗剪
仅营销叫卖
无实物展示（围绕商品讲解）
内容与商品无关推广
穿帮 / 不符现实
擦边低俗
恶心恐怖
盗版内容
内容介绍不详细（低质）
站外引流交易
静止帧

Allowed tagsAttribute values:
视频不可见

Mapping:
video_product_mismatch -> 挂车商品与讲解商品不一致
misleading_functionality_or_effect -> 功能效果虚假夸大
suspected_pirated_or_reused_content -> 疑似盗版 / 疑似盗剪
pure_marketing_pitch -> 仅营销叫卖
no_physical_product_display -> 无实物展示（围绕商品讲解）
irrelevant_product_promotion -> 内容与商品无关推广
unrealistic_or_continuity_error -> 穿帮 / 不符现实
sexual_or_vulgar_hook -> 擦边低俗
disgusting_or_terrifying_visual -> 恶心恐怖
pirated_content -> 盗版内容
description_not_detailed -> 内容介绍不详细（低质）
out_of_app_transaction -> 站外引流交易
still_frame -> 静止帧
none -> no tagsContent

Output JSON schema:
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

Output rules:
- human_review_labels.ratings must be ["video"] when tagsContent is non-empty.
- human_review_labels.ratings must be ["ok"] when tagsContent is empty.
- Do not output tagsProduct.
- Do not output manual_review.
- Do not output old issue types such as misleading_functionality_and_effect, inconsistent_product_promotion, potential_pirated, no_physical_product_display from old schemas, description_not_detailed from old schemas, or only_marketing_sales_pitches.
- If video frames are missing or unusable, include 视频不可见 in tagsAttribute and do not claim video-frame evidence.
- If no issue is supported by concrete evidence, set overall_decision=clean, primary_issue_type=none, detected_issue_types=[], ratings=["ok"], tagsContent=[].
```

- [ ] **Step 3: Verify prompt constraints**

Run:

```bash
rg -n "manual_review|tagsProduct|product_review_summary|comments|seller_id_str|product_id|url" /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_*.txt
```

Expected: occurrences only appear in explicit "do not use" or "do not output" instructions.

## Task 7: Rewrite Rule Files To Video-First v13

**Files:**
- Modify: `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json`
- Modify: `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt`

- [ ] **Step 1: Update structured rules to v13**

The structured JSON must have:

```json
{
  "version": "2026-09-17-v13-video-first-binary-boundary-calibration",
  "project": "Video_product_lowquality",
  "decision_values": ["problematic", "clean"],
  "evidence_contract": {
    "judgment_evidence": ["frame_list", "ASR", "OCR", "images", "country"],
    "display_only": ["url", "product_id", "seller_id_str", "comments", "product_review_summary"]
  },
  "primary_issue_types": [
    "video_product_mismatch",
    "misleading_functionality_or_effect",
    "suspected_pirated_or_reused_content",
    "pure_marketing_pitch",
    "no_physical_product_display",
    "irrelevant_product_promotion",
    "unrealistic_or_continuity_error",
    "sexual_or_vulgar_hook",
    "disgusting_or_terrifying_visual",
    "pirated_content",
    "description_not_detailed",
    "out_of_app_transaction",
    "still_frame",
    "none"
  ],
  "rules": []
}
```

Fill `rules` with one object per issue type using the boundary text from the approved spec. Do not include product quality, product-side mismatch, product review summary, comment review, or tagsProduct rules.

- [ ] **Step 2: Update text rules to v13**

The text rule file must start with:

```text
Video Product Lowquality Rules v13

This workflow is video-first. The final model judges only video low-quality signals.

Allowed model evidence: frame_list, ASR, OCR, images, country.
Display-only fields: url, product_id, seller_id_str, comments, product_review_summary.
Do not use display-only fields as judgment evidence.

Allowed final decisions: problematic, clean.
Forbidden final decision: manual_review.
Forbidden output: tagsProduct.
```

Then include each issue boundary from the approved spec.

- [ ] **Step 3: Mirror rule files**

Copy the repo rule files to the wiki mirror:

```bash
cp /Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json /Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json
cp /Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt /Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt
```

- [ ] **Step 4: Validate JSON and diff mirror**

Run:

```bash
python3 -m json.tool /Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json >/tmp/video_lowquality_rules_check.json
diff -u /Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json /Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json
diff -u /Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt /Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt
```

Expected: `json.tool` succeeds; both `diff` commands produce no output.

## Task 8: Remove Old Workflow Dependencies From Tests And Docs

**Files:**
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/Video-Product-Lowquality-Current-State.md`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/Video-Product-Lowquality-Workflow.md`
- Modify: `/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/Video-Product-Lowquality-Output-Schema.md`

- [ ] **Step 1: Update tests that assert old product/comment behavior**

Remove or rewrite tests whose expected behavior requires:

- `product_review_summary` as model evidence;
- `comments` as model evidence;
- `manual_review`;
- `tagsProduct`;
- `Output_Validator` as an active node.

Keep tests for URL parsing helpers if still used by `frame_list` or `images` parsing.

- [ ] **Step 2: Update current-state docs**

Replace old current-state claims with:

```text
Current workflow target: video-first binary low-quality review.
Final model evidence: frame_list, ASR, OCR, images, country.
Display-only fields: url, product_id, seller_id_str, comments, product_review_summary.
Final decisions: problematic, clean.
Removed from active path: Comment_Review_Signal_Builder, Context_Builder, Output_Validator.
```

- [ ] **Step 3: Run documentation consistency search**

Run:

```bash
rg -n "manual_review|tagsProduct|product_review_summary|Comment_Review_Signal_Builder|Output_Validator|Context_Builder" /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality /Users/bytedance/Desktop/EU-LLM_Wiki/projects/video-product-lowquality /Users/bytedance/video-product-lowquality/rules
```

Expected: remaining hits are historical notes or explicit "removed/display-only/do not use" statements.

## Task 9: Final Local Verification

**Files:**
- All files modified in Tasks 1-8

- [ ] **Step 1: Run unit tests**

Run:

```bash
python3 -m unittest /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/tests/test_code_nodes.py
```

Expected: all tests pass.

- [ ] **Step 2: Run schema/key search**

Run:

```bash
rg -n "manual_review|tagsProduct|poor_product_quality|product_side_mismatch|product_review_summary|comments|seller_id_str|product_id|url" /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes /Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts /Users/bytedance/video-product-lowquality/rules
```

Expected: `manual_review`, `tagsProduct`, product-side issue names, `comments`, and `product_review_summary` appear only in explicit forbidden/display-only text, not in code-node input extraction or LLM evidence sections.

- [ ] **Step 3: Check git diff**

Run:

```bash
git -C /Users/bytedance/video-product-lowquality status --short
git -C /Users/bytedance/video-product-lowquality diff --stat
```

Expected: rule files and plan/spec docs are visible in the git repo; `.superpowers/` remains untracked and is not staged.

## Task 10: Commit And Push Rule Source

**Files:**
- `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_structured_v1.json`
- `/Users/bytedance/video-product-lowquality/rules/video_product_lowquality_rules_text_v1.txt`
- `/Users/bytedance/video-product-lowquality/docs/superpowers/plans/2026-09-17-video-first-lowquality-workflow.md`

- [ ] **Step 1: Stage only repo-owned files**

Run:

```bash
git -C /Users/bytedance/video-product-lowquality add rules/video_product_lowquality_rules_structured_v1.json rules/video_product_lowquality_rules_text_v1.txt docs/superpowers/plans/2026-09-17-video-first-lowquality-workflow.md
```

- [ ] **Step 2: Commit**

Run:

```bash
git -C /Users/bytedance/video-product-lowquality commit -m "Calibrate video-first lowquality workflow"
```

Expected: commit succeeds. `.superpowers/` remains untracked.

- [ ] **Step 3: Push**

Run:

```bash
git -C /Users/bytedance/video-product-lowquality push origin main
```

Expected: push succeeds.

- [ ] **Step 4: Verify raw URLs**

Run:

```bash
curl -fsSL https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_structured_v1.json | python3 -m json.tool >/tmp/video_lowquality_raw_rules.json
curl -fsSL https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_text_v1.txt | head -5
```

Expected: JSON validates; text output starts with `Video Product Lowquality Rules v13`.

## Task 11: Write Manual Aicolate Update Guide

**Files:**
- Create: `/Users/bytedance/video-product-lowquality/docs/aicolate-manual-update-2026-09-17.md`

- [ ] **Step 1: Create the guide**

The guide must include sections in this order:

```markdown
# Aicolate Manual Update Guide - Video Product Lowquality v13

## Step 0: Do Not Change These Display Fields

Keep url, product_id, seller_id_str, comments, and product_review_summary available only for human display or tracing. Do not wire them into the final LLM prompt.

## Step 1: HTTP Rule Nodes

Use the same raw URLs:

- https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_structured_v1.json
- https://raw.githubusercontent.com/danielwanyan/video-product-lowquality/main/rules/video_product_lowquality_rules_text_v1.txt

Confirm structured rule version is 2026-09-17-v13-video-first-binary-boundary-calibration.

## Step 2: Video_Content_Builder

Input Variables:
- frame_list = {{frame_list}}
- ASR = {{ASR}}
- OCR = {{OCR}}
- video_id = {{video_id}}

Output panel:
- video_frame_urls
- video_frame_manifest
- video_text_panel
- video_signal_flags_json
- video_data_quality_json

Paste full code from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/video_content_builder.py

Stop here and run one sample before continuing.

## Step 3: Product_Image_Aux_Builder

Input Variables:
- images = {{images}}
- country = {{country}}

Output panel:
- product_image_urls
- product_image_manifest
- product_aux_panel
- product_aux_data_quality_json

Paste full code from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/product_image_aux_builder.py

Stop here and run one sample before continuing.

## Step 4: Boundary_Attention_Builder

Input Variables:
- video_text_panel = {{video_text_panel}}
- video_data_quality_json = {{video_data_quality_json}}
- product_aux_panel = {{product_aux_panel}}
- product_aux_data_quality_json = {{product_aux_data_quality_json}}
- rules_json_body = {{project_rules_json_fetch.body}}
- rules_text_body = {{project_rules_text_fetch.body}}

Output panel:
- boundary_attention_packet
- active_boundary_groups
- must_not_infer
- positive_hit_tests
- allowed_issue_types

Paste full code from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/boundary_attention_builder.py

Stop here and run one sample before continuing.

## Step 5: Evidence_Gate

Input Variables:
- video_frame_urls = {{video_frame_urls}}
- video_frame_manifest = {{video_frame_manifest}}
- video_data_quality_json = {{video_data_quality_json}}
- product_image_urls = {{product_image_urls}}
- product_image_manifest = {{product_image_manifest}}
- product_aux_data_quality_json = {{product_aux_data_quality_json}}
- boundary_attention_packet = {{boundary_attention_packet}}
- allowed_issue_types = {{allowed_issue_types}}

Output panel:
- gated_all_image_urls
- gated_all_image_manifest
- evidence_gate_panel
- forbidden_claims
- allowed_issue_types
- final_reviewer_context

Paste full code from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/code_nodes/evidence_gate.py

Stop here and run one sample before continuing.

## Step 6: Unified_Video_Lowquality_Reviewer

System Prompt:
Paste full prompt from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_system_prompt.txt

User Prompt:
Paste full prompt from:
/Users/bytedance/Desktop/EU-LLM_Wiki/wiki/projects/video-product-lowquality/prompts/unified_multimodal_risk_reviewer_user_prompt.txt

Input Variables:
- gated_all_image_urls = {{gated_all_image_urls}}
- gated_all_image_manifest = {{gated_all_image_manifest}}
- final_reviewer_context = {{final_reviewer_context}}
- evidence_gate_panel = {{evidence_gate_panel}}
- allowed_issue_types = {{allowed_issue_types}}
- forbidden_claims = {{forbidden_claims}}
- video_id = {{video_id}}
- ASR = {{ASR}}
- OCR = {{OCR}}
- country = {{country}}

Do not wire url, product_id, seller_id_str, comments, or product_review_summary.

## Step 7: End Node

Connect End directly to the single output field from Unified_Video_Lowquality_Reviewer.

Remove Output_Validator from the active path.

## Step 8: Smoke Test

Run three samples:
- a known clean color-only mismatch false positive;
- a known problematic MFE sample;
- a known frame-missing sample if available.

Expected:
- no manual_review;
- no tagsProduct;
- ratings is ["video"] for issues and ["ok"] for clean;
- comments and product_review_summary are not referenced in final_reason_cn.
```

- [ ] **Step 2: Verify guide does not include stale wiring**

Run:

```bash
rg -n "Comment_Review_Signal_Builder|Context_Builder|Output_Validator|product_review_summary|comments|manual_review|tagsProduct" /Users/bytedance/video-product-lowquality/docs/aicolate-manual-update-2026-09-17.md
```

Expected: hits only appear in explicit "do not wire", "remove", or "not allowed" instructions.

## Task 12: Handoff To User

**Files:**
- `/Users/bytedance/video-product-lowquality/docs/aicolate-manual-update-2026-09-17.md`

- [ ] **Step 1: Summarize completed artifacts**

Report:

- local files changed;
- tests run and results;
- Git commit and push status;
- raw URL verification result;
- first Aicolate node the user should update.

- [ ] **Step 2: Start one-node-at-a-time Aicolate instructions**

Give only Step 1 and Step 2 from the manual update guide first:

1. HTTP rule node verification.
2. `Video_Content_Builder` full code and Output panel fields.

Then stop and wait for user confirmation before giving the next node.
