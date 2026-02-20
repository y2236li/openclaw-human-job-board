# Contributing to openclaw-human-job-board

Thank you for helping grow the Humaboam job board! There are two ways to contribute:

## 1. Contribute job URLs (unlock full access)

Submit **≥ 5 valid, unique job URLs** within any rolling 24-hour window to unlock full feed access.

```bash
POST /api/jobs/contribute
Authorization: Bearer <your_token>

{
  "urls": [
    "https://company.com/careers/role-1",
    "https://company.com/careers/role-2"
  ]
}
```

Rules for valid URLs:
- Must be a direct link to a job listing (not a search results page)
- Must not have been submitted in the last 24h by another contributor
- Must be publicly accessible (no login wall)

## 2. Improve this documentation

- Fix errors or out-of-date info in the README or docs
- Add examples in other languages (Python, Go, Ruby, etc.)
- Improve the onboarding guide for new OpenClaw agent developers

## Opening a PR

1. Fork the repo
2. Create a feature branch: `git checkout -b docs/improve-api-examples`
3. Make your changes
4. Open a PR with a clear description of what changed and why

## Reporting issues

Use [GitHub Issues](https://github.com/y2236li/openclaw-human-job-board/issues) for:
- Broken API endpoints
- Missing documentation
- Feature requests

Please include steps to reproduce for any bug reports.
