# Workday jobs API: read any myworkdayjobs.com career site, or search 417k Workday jobs in one call

Workday has no public jobs API with a key and docs. But every career site hosted on `*.myworkdayjobs.com` loads its job list from a public JSON endpoint, the same one the page itself calls. This guide shows how that endpoint behaves, the four things that break naive scrapers, and two ready-made ways to get the data without maintaining the crawler yourself.

## The endpoint

A career site URL looks like `https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite`: tenant `nvidia`, data center `wd5`, site `NVIDIAExternalCareerSite`. Its job list is:

```
POST https://nvidia.wd5.myworkdayjobs.com/wday/cxs/nvidia/NVIDIAExternalCareerSite/jobs
content-type: application/json

{"appliedFacets": {}, "limit": 20, "offset": 0, "searchText": ""}
```

and one job, with its description, is a GET on `/wday/cxs/<tenant>/<site><externalPath>`.

## What breaks

1. **Page size is 20.** `limit: 100` answers HTTP 400.
2. **`total` is only right on the first page.** Later pages return `total: 0`.
3. **A list never goes past 2,000 jobs.** A site with more reports `total: 2000` and stops there. To get the rest you have to split the list by a facet (location, job category...) with `appliedFacets` and merge the parts. NVIDIA reported 2,000 and had 2,472 on 2026-11-16.
4. **Paging past the end does not return an empty page.** It wraps and serves rows again, so a loop that waits for an empty page duplicates data.

Also: there is no exact posting date, only "Posted Today", "Posted 3 Days Ago" or "Posted 30+ Days Ago"; the list has no salary and no department; and many requests from one address in a short time get HTTP 429.

## Option 1: one or several career sites, with descriptions

The Apify Actor [Workday Jobs Scraper](https://apify.com/renzomacar/workday-jobs-scraper) takes career site URLs and handles the four points above, including the split by facets for sites over 2,000 jobs.

```json
{
  "companies": ["https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite"],
  "searchText": "deep learning",
  "maxItems": 5
}
```

Run on 2026-11-16: 5 jobs with descriptions in 5.7 seconds. One record (description cut here; full file: [sample-single-site.json](sample-single-site.json)):

```json
{
  "source": "workday",
  "companyToken": "nvidia",
  "companyName": "CN05 NVIDIA Shanghai WFOE",
  "jobId": "ac092cf22d13102ce18164796f0d0000",
  "title": "Deep Learning Performance Architect",
  "department": null,
  "team": null,
  "employmentType": "FULL_TIME",
  "location": "China, Shanghai",
  "locations": [
    "China, Shanghai"
  ],
  "country": "China",
  "isRemote": false,
  "workplaceType": null,
  "remoteSource": "unknown",
  "salaryMin": null,
  "salaryMax": null,
  "salaryCurrency": null,
  "salaryInterval": null,
  "salaryText": null,
  "salarySource": null,
  "postedAt": "2026-07-27T00:00:00.000Z",
  "updatedAt": null,
  "jobUrl": "https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite/job/China-Shanghai/Deep-Learning-Performance-Architect_JR2021807",
  "applyUrl": "https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite/job/China-Shanghai/Deep-Learning-Performance-Architect_JR2021807/apply",
  "descriptionText": "NVIDIA is developing GPU and system architectures that accelerate deep learning and high-performance computing applications. We are looking for an expert deep l...",
  "descriptionHtml": null,
  "scrapedAt": "2026-11-16T20:35:53.228Z",
  "requisitionId": "JR2021807",
  "departments": [],
  "postedOnText": "Posted 30+ Days Ago",
  "postedAtEstimated": "2026-10-17T20:35:53.228Z",
  "canApply": true,
  "tenant": "nvidia",
  "careerSite": "NVIDIAExternalCareerSite",
  "boardUrl": "https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite"
}
```

## Option 2: search across 880 Workday employers at once

If you do not know which companies to ask, the Actor [ATS Jobs Search](https://apify.com/renzomacar/ats-jobs-search) keeps an index of the open jobs of 880 Workday employers (417,945 jobs on 2026-11-16), plus 20,000 companies on Greenhouse, Lever, Ashby, Workable, SmartRecruiters, Personio, Recruitee, Teamtailor and Breezy HR. The index is rebuilt twice a day, so a search answers in seconds.

```json
{
  "ats": ["workday"],
  "keywords": ["data engineer"],
  "postedWithinDays": 3,
  "maxItems": 25
}
```

Run on 2026-11-16: 25 jobs in 6.1 seconds. First twelve (full file: [sample-search.json](sample-search.json)):

| Company | Title | Location | Posted |
|---|---|---|---|
| geico | Business Intelligence Analyst / Data Engineer | Remote (United States) | 2026-11-16 |
| jll | Lead Engineer, Data Centers | LaGrange,GA | 2026-11-16 |
| jll | Building Engineer, Data Centers- Night Shift | Amarillo TX (+1 more) | 2026-11-16 |
| jll | Building Engineer, Data Centers | Zebulon, NC | 2026-11-16 |
| jll | Building Engineer, Data Centers | Claude TX (+1 more) | 2026-11-16 |
| CrowdStrike | Sr. Network Engineer - Data Center and Cloud (Hybrid, London) | United Kingdom - London | 2026-11-16 |
| General Motors | Staff Software Engineer - Data | Sunnyvale, California, United States of  | 2026-11-16 |
| Stryker | Senior Engineer, EDT Data & Analytics Cloud Engineering | Gurugram, India | 2026-11-16 |
| Blue Origin | Senior Software Engineer, Accelerated Computing Runtime, Project Sunri | Denver CO (+1 more) | 2026-11-16 |
| NVIDIA | Senior Data Application Engineer – Enterprise Data Management | US, CA, Santa Clara | 2026-11-16 |
| Cisco | Solution Test Engineer - 8+yrs - Python, PyATS, Switching/Routing/Data | Bangalore, India | 2026-11-16 |
| Cisco | Solution Test Engineer - 5+yrs - Python, PyATS, Switching/Routing/Data | Bangalore, India | 2026-11-16 |

A keyword matches when all of its words are in the title, in any order, which is why "Engineer, Data Centers" is in the list. Put the words in quotes inside the entry (`"\"data engineer\""`) to match the exact phrase. About half of the Workday employers show their display name; the rest show their Workday id (`jll`, `geico`).

## Call it from code

You need an Apify account and its API token (`APIFY_TOKEN`).

```bash
curl -X POST "https://api.apify.com/v2/acts/renzomacar~workday-jobs-scraper/run-sync-get-dataset-items?token=$APIFY_TOKEN"   -H "content-type: application/json"   -d '{"companies":["https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite"],"maxItems":50}'
```

```python
from apify_client import ApifyClient

client = ApifyClient("APIFY_TOKEN")
run = client.actor("renzomacar/ats-jobs-search").call(run_input={
    "ats": ["workday"],
    "keywords": ["data engineer"],
    "postedWithinDays": 3,
    "maxItems": 100,
})
for job in client.dataset(run["defaultDatasetId"]).iterate_items():
    print(job["company"], job["title"], job["url"])
```

## Limits

- Only public career sites are read: no login, no candidate data.
- The search index covers 880 Workday employers, not every company on Workday. For a company outside the list, use Option 1 with its career site URL.
- Workday publishes no salary field in the job list, so Workday jobs never match salary filters.
- `postedAt` is a day, derived from "Posted N Days Ago", and is empty for jobs older than 30 days.
- Both Actors are paid per result on Apify; prices are on their pages.
