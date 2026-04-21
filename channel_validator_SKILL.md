# Channel Validator Skill (Python)

This Python script validates marketing assets for platform-specific requirements including:
- Social media character limits
- Email subject and preview length
- CTA presence
- Basic content readiness checks

## Python Code

```python
from typing import Any, Dict, List
import re

SOCIAL_LIMITS = {
    "x": 280,
    "twitter": 280,
    "instagram": 2200,
    "linkedin": 3000
}

EMAIL_RULES = {
    "subject_max": 60,
    "preview_max": 90
}

CTA_PATTERNS = [
    r"\bapply now\b",
    r"\blearn more\b",
    r"\bget started\b",
    r"\bopen an account\b",
    r"\bsign up\b",
    r"\bjoin now\b",
    r"\bsee how\b",
    r"\bcheck it out\b",
    r"\bstart today\b",
    r"\bcompare options\b"
]


def _text(value: Any) -> str:
    return "" if value is None else str(value).strip()


def _char_count(text: str) -> int:
    return len(text)


def _word_count(text: str) -> int:
    return len(text.split()) if text else 0


def _has_cta(text: str) -> bool:
    lowered = text.lower()
    return any(re.search(pattern, lowered) for pattern in CTA_PATTERNS)


def _status_from_issues(issues: List[str]) -> str:
    return "pass" if not issues else "fail"


def validate_social_post(post: Dict[str, Any]) -> Dict[str, Any]:
    platform = _text(post.get("platform"))
    text = _text(post.get("text"))
    platform_key = platform.lower()

    issues = []
    recommendations = []

    limit = SOCIAL_LIMITS.get(platform_key)
    length = _char_count(text)

    if limit is None:
        return {
            "asset_type": "social_post",
            "asset_label": _text(post.get("label")) or f"{platform.lower()}_post",
            "platform": platform,
            "status": "warning",
            "issues": [f"Unsupported platform: {platform}"],
            "recommendations": ["Use X, Instagram, or LinkedIn."],
            "metrics": {
                "character_count": length,
                "word_count": _word_count(text),
                "cta_detected": _has_cta(text)
            }
        }

    if length > limit:
        issues.append(f"Exceeds {platform} character limit by {length - limit} characters.")
        recommendations.append(f"Reduce copy to {limit} characters or fewer.")

    if not _has_cta(text):
        issues.append("No clear CTA detected.")
        recommendations.append("Add a CTA like 'Learn more', 'Apply now', or 'Get started'.")

    return {
        "asset_type": "social_post",
        "asset_label": _text(post.get("label")) or f"{platform.lower()}_post",
        "platform": platform,
        "status": _status_from_issues(issues),
        "issues": issues,
        "recommendations": recommendations,
        "metrics": {
            "character_count": length,
            "character_limit": limit,
            "characters_remaining": limit - length,
            "word_count": _word_count(text),
            "cta_detected": _has_cta(text)
        }
    }


def validate_email(email: Dict[str, Any]) -> Dict[str, Any]:
    label = _text(email.get("label")) or "email"
    subject = _text(email.get("subject"))
    preview_text = _text(email.get("preview_text"))
    body = _text(email.get("body"))
    cta = _text(email.get("cta"))

    issues = []
    recommendations = []

    subject_length = _char_count(subject)
    preview_length = _char_count(preview_text)
    combined_text = f"{body} {cta}".strip()

    if subject_length > EMAIL_RULES["subject_max"]:
        issues.append(
            f"Subject line exceeds recommended max by {subject_length - EMAIL_RULES['subject_max']} characters."
        )
        recommendations.append("Shorten the subject line for stronger mobile readability.")

    if preview_length > EMAIL_RULES["preview_max"]:
        issues.append(
            f"Preview text exceeds recommended max by {preview_length - EMAIL_RULES['preview_max']} characters."
        )
        recommendations.append("Shorten preview text to improve inbox display.")

    if not body:
        issues.append("Email body is missing.")
        recommendations.append("Add body copy before routing for review.")

    if not cta and not _has_cta(body):
        issues.append("No clear CTA detected.")
        recommendations.append("Add a CTA such as 'Apply now' or 'Learn more'.")

    return {
        "asset_type": "email",
        "asset_label": label,
        "status": _status_from_issues(issues),
        "issues": issues,
        "recommendations": recommendations,
        "metrics": {
            "subject_length": subject_length,
            "subject_recommended_max": EMAIL_RULES["subject_max"],
            "preview_length": preview_length,
            "preview_recommended_max": EMAIL_RULES["preview_max"],
            "body_character_count": _char_count(body),
            "body_word_count": _word_count(body),
            "cta_detected": bool(cta) or _has_cta(combined_text)
        }
    }


def validate_web_section(section_name: str, section: Dict[str, Any]) -> Dict[str, Any]:
    issues = []
    recommendations = []

    parts = []
    for _, value in section.items():
        if isinstance(value, str):
            parts.append(value.strip())

    combined_text = " ".join([p for p in parts if p]).strip()

    if not combined_text:
        issues.append(f"{section_name} is empty.")
        recommendations.append(f"Add content to {section_name}.")

    if "cta" in section_name.lower() and not _has_cta(combined_text):
        issues.append("No clear CTA detected.")
        recommendations.append("Add a CTA like 'Get started' or 'Apply now'.")

    return {
        "asset_type": "web_section",
        "asset_label": section_name,
        "status": _status_from_issues(issues),
        "issues": issues,
        "recommendations": recommendations,
        "metrics": {
            "character_count": _char_count(combined_text),
            "word_count": _word_count(combined_text),
            "cta_detected": _has_cta(combined_text)
        }
    }


def summarize(results: List[Dict[str, Any]]) -> Dict[str, Any]:
    passed = sum(1 for item in results if item["status"] == "pass")
    failed = sum(1 for item in results if item["status"] == "fail")
    warnings = sum(1 for item in results if item["status"] == "warning")

    return {
        "total_assets_checked": len(results),
        "passed": passed,
        "failed": failed,
        "warnings": warnings,
        "overall_status": "pass" if failed == 0 else "fail"
    }


def main(payload: Dict[str, Any]) -> Dict[str, Any]:
    results = []

    for post in payload.get("social_posts", []):
        results.append(validate_social_post(post))

    for email in payload.get("emails", []):
        results.append(validate_email(email))

    for section_name, section in payload.get("web_content", {}).items():
        if isinstance(section, dict):
            results.append(validate_web_section(section_name, section))

    return {
        "summary": summarize(results),
        "results": results
    }
```

## Usage

Pass a structured payload containing:
- social_posts
- emails
- web_content

The script returns:
- summary (pass/fail counts)
- detailed validation results per asset
