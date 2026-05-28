---
name: zxgk-court-execution-query
description: Query 中国执行信息公开网 / 全国法院信息综合查询 at zxgk.court.gov.cn for executed-person cases, detail pages, execution courts, execution amounts, and amount totals. Use real Chromium/Edge, page captcha, searchZhcx.do pagination, hidden iframe detail extraction, duplicate caseCode handling, and explicit missing-amount reporting.
---

# Hermes Skill: 中国执行信息公开网执行案件查询与金额汇总

## When To Use

Use this skill when the user asks for any of the following:

- Query 中国执行信息公开网.
- Query 全国法院信息综合查询.
- Query `https://zxgk.court.gov.cn/zhzxgk/`.
- Find executed-person cases for a company or individual.
- Extract `案号`, `立案时间`, `执行法院`, `执行标的`.
- Sum execution amounts.
- Summarize execution amount by court.

## Non-Negotiable Rules

1. Do not use plain `curl`, `requests`, or raw HTTP as the primary browser.
   The site uses JS challenge, browser checks, captcha, cookies, and form-bound state.

2. Use real Chromium or Edge.
   On Linux, prefer Playwright Chromium with `headless=False` under `xvfb-run` when no desktop exists.

3. Read captcha from the currently displayed page element.
   Do not re-request `captcha.do`; that can refresh server-side captcha state.

4. Use `searchZhcx.do` only for the list.
   The list does not reliably contain `执行法院` or `执行标的`.

5. Use `detailZhcx.do` for detail fields.
   Detail pages contain `执行法院` and `执行标的`.

6. Submit detail requests through the page's existing form into a hidden same-origin iframe.
   This preserves the original query page and captcha state.

7. Deduplicate final case count by `caseCode`.
   Keep every `dePartyCardNum` for detail fetching because duplicate `caseCode` rows may differ by encrypted card value.

8. If a detail page returns `页面不存在`, retry the same `caseCode` with every available `dePartyCardNum`.
   Only mark the case as `NO_DETAIL_AMOUNT` after all variants fail.

9. If any case has no retrievable amount, report `已取到金额的案件合计`, not `全部案件总金额`.

## Linux Setup

Install Playwright:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install playwright
python -m playwright install chromium
```

For headless Linux servers:

```bash
sudo apt-get update
sudo apt-get install -y xvfb
xvfb-run -a python query_zxgk.py
```

Recommended browser launch:

```python
browser = p.chromium.launch(headless=False)
```

If there is no desktop, run the whole script under `xvfb-run -a`.

## Main Page

Open:

```text
https://zxgk.court.gov.cn/zhzxgk/
```

Wait for:

```text
#pName
#pCardNum
#captchaImg
#captchaId
#searchFormnewdetail
```

If these elements do not appear, the browser did not pass the page challenge. Restart the browser session, use non-headless mode, and wait longer.

## Captcha Workflow

Screenshot the current captcha element:

```python
page.locator("#captchaImg").screenshot(path="captcha.png")
```

Alternative in-page canvas export:

```javascript
const img = document.querySelector("#captchaImg");
const canvas = document.createElement("canvas");
canvas.width = img.naturalWidth || img.width;
canvas.height = img.naturalHeight || img.height;
canvas.getContext("2d").drawImage(img, 0, 0);
const base64 = canvas.toDataURL("image/png").split(",")[1];
```

Verify recognized captcha:

```javascript
async function checkCaptcha(code) {
  const captchaId = document.querySelector("#captchaId").value;
  const res = await fetch(
    "checkyzm?captchaId=" + encodeURIComponent(captchaId) +
    "&pCode=" + encodeURIComponent(code),
    { credentials: "include" }
  );
  return (await res.text()).trim();
}
```

Meaning:

```text
1 = correct
0 = incorrect
```

Do not continue until captcha check returns `1`.

## Search List

Endpoint:

```text
searchZhcx.do
```

Run from page context:

```javascript
async function searchPage(companyName, captchaCode, pageNumber) {
  const params = new URLSearchParams();
  params.set("pName", companyName);
  params.set("pCardNum", "");
  params.set("selectCourtId", "0");
  params.set("pCode", captchaCode);
  params.set("captchaId", document.querySelector("#captchaId").value);
  params.set("searchCourtName", "全国法院（包含地方各级法院）");
  params.set("selectCourtArrange", "1");
  params.set("currentPage", String(pageNumber));

  const res = await fetch("searchZhcx.do", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"
    },
    body: params.toString(),
    credentials: "include"
  });

  return await res.json();
}
```

Expected response shape:

```javascript
[
  {
    totalSize: 122,
    totalPage: 13,
    currentPage: 1,
    result: [...]
  }
]
```

Save from each row:

```text
pname
caseCode
caseCreateTime
dePartyCardNum
```

## Duplicate Case Handling

Group list rows by `caseCode`.

For reporting case count:

```text
unique case count = count(distinct caseCode)
```

For fetching details:

```text
try every row under the same caseCode, because dePartyCardNum may differ
```

Never throw away duplicate rows before detail fetching.

## Detail Fetching

Endpoint:

```text
detailZhcx.do
```

Do not directly `fetch("detailZhcx.do")` first. Use the site's form:

```text
#searchFormnewdetail
```

Create hidden iframe:

```javascript
function ensureDetailFrame() {
  let iframe = document.querySelector("#detailFrame");
  if (!iframe) {
    iframe = document.createElement("iframe");
    iframe.name = "detailFrame";
    iframe.id = "detailFrame";
    iframe.style.display = "none";
    document.body.appendChild(iframe);
  }
}
```

Submit detail form:

```javascript
function submitDetail(row, captchaCode) {
  ensureDetailFrame();

  const form = document.querySelector("#searchFormnewdetail");
  form.target = "detailFrame";

  document.querySelector("#pnameNewDel").value = row.pname;
  document.querySelector("#cardNumNewDel").value = "";
  document.querySelector("#j_captchaNewDel").value = captchaCode;
  document.querySelector("#caseCodeNewDel").value = row.caseCode;
  document.querySelector("#dePartyCardNumNewDel").value = row.dePartyCardNum;
  document.querySelector("#captchaIdNewDel").value =
    document.querySelector("#captchaId").value;

  form.submit();
}
```

Poll iframe:

```javascript
async function readDetailText() {
  for (let i = 0; i < 30; i++) {
    await new Promise(resolve => setTimeout(resolve, 500));
    const text =
      document.querySelector("#detailFrame")
        ?.contentDocument
        ?.body
        ?.innerText || "";

    if (text.includes("执行标的") || text.includes("页面不存在")) {
      return text;
    }
  }
  return "";
}
```

## Detail Parsing

Detail text labels:

```text
被执行人姓名/名称：
身份证号码/组织机构代码：
执行法院：
立案时间：
案号：
执行标的：
```

JavaScript parser:

```javascript
function parseDetailText(text) {
  function pick(label) {
    const re = new RegExp(label + "[：:]\\s*([^\\n]+)");
    const match = text.match(re);
    return match ? match[1].trim() : "";
  }

  return {
    name: pick("被执行人姓名/名称"),
    idCode: pick("身份证号码/组织机构代码"),
    execCourt: pick("执行法院"),
    filingDate: pick("立案时间"),
    caseCode: pick("案号"),
    execMoney: pick("执行标的"),
    rawText: text
  };
}
```

Amount parser:

```javascript
function parseAmount(value) {
  value = String(value || "").trim();
  if (/^\d+(\.\d+)?$/.test(value)) return Number(value);
  return null;
}
```

Python amount parser:

```python
from decimal import Decimal
import re

def parse_amount(value):
    value = (value or "").strip()
    if re.match(r"^\d+(\.\d+)?$", value):
        return Decimal(value)
    return None
```

## End-To-End Python Skeleton

```python
from playwright.sync_api import sync_playwright
from decimal import Decimal
from collections import defaultdict
import re
import csv

COMPANY = "中国一冶集团有限公司"

def parse_amount(value):
    value = (value or "").strip()
    if re.match(r"^\d+(\.\d+)?$", value):
        return Decimal(value)
    return None

def main():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False)
        page = browser.new_page()
        page.goto("https://zxgk.court.gov.cn/zhzxgk/", wait_until="networkidle")

        page.wait_for_selector("#captchaImg")
        page.locator("#captchaImg").screenshot(path="captcha.png")

        code = input("Input captcha from captcha.png: ").strip()

        ok = page.evaluate("""
        async (code) => {
          const captchaId = document.querySelector("#captchaId").value;
          const res = await fetch(
            "checkyzm?captchaId=" + encodeURIComponent(captchaId) +
            "&pCode=" + encodeURIComponent(code),
            { credentials: "include" }
          );
          return (await res.text()).trim();
        }
        """, code)

        if ok != "1":
            raise RuntimeError("captcha check failed")

        def search_page(page_no):
            return page.evaluate("""
            async ({ company, code, pageNo }) => {
              const params = new URLSearchParams();
              params.set("pName", company);
              params.set("pCardNum", "");
              params.set("selectCourtId", "0");
              params.set("pCode", code);
              params.set("captchaId", document.querySelector("#captchaId").value);
              params.set("searchCourtName", "全国法院（包含地方各级法院）");
              params.set("selectCourtArrange", "1");
              params.set("currentPage", String(pageNo));

              const res = await fetch("searchZhcx.do", {
                method: "POST",
                headers: {
                  "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"
                },
                body: params.toString(),
                credentials: "include"
              });
              return await res.json();
            }
            """, {"company": COMPANY, "code": code, "pageNo": page_no})

        first = search_page(1)
        total_page = first[0]["totalPage"]
        all_rows = list(first[0]["result"])

        for page_no in range(2, total_page + 1):
            data = search_page(page_no)
            all_rows.extend(data[0]["result"])

        by_case = defaultdict(list)
        for row in all_rows:
            by_case[row["caseCode"]].append(row)

        details = []

        for case_code, rows in by_case.items():
            found = None

            for row in rows:
                text = page.evaluate("""
                async ({ row, code }) => {
                  let iframe = document.querySelector("#detailFrame");
                  if (!iframe) {
                    iframe = document.createElement("iframe");
                    iframe.name = "detailFrame";
                    iframe.id = "detailFrame";
                    iframe.style.display = "none";
                    document.body.appendChild(iframe);
                  }

                  const form = document.querySelector("#searchFormnewdetail");
                  form.target = "detailFrame";

                  document.querySelector("#pnameNewDel").value = row.pname;
                  document.querySelector("#cardNumNewDel").value = "";
                  document.querySelector("#j_captchaNewDel").value = code;
                  document.querySelector("#caseCodeNewDel").value = row.caseCode;
                  document.querySelector("#dePartyCardNumNewDel").value = row.dePartyCardNum;
                  document.querySelector("#captchaIdNewDel").value =
                    document.querySelector("#captchaId").value;

                  form.submit();

                  for (let i = 0; i < 30; i++) {
                    await new Promise(resolve => setTimeout(resolve, 500));
                    const text =
                      document.querySelector("#detailFrame")
                        ?.contentDocument
                        ?.body
                        ?.innerText || "";

                    if (text.includes("执行标的") || text.includes("页面不存在")) {
                      return text;
                    }
                  }
                  return "";
                }
                """, {"row": row, "code": code})

                if "执行标的" in text:
                    found = text
                    break

            if not found:
                details.append({
                    "CaseCode": case_code,
                    "FilingDate": "",
                    "ExecCourt": "",
                    "ExecMoney": "",
                    "Amount": None,
                    "DetailStatus": "NO_DETAIL_AMOUNT",
                    "RawText": ""
                })
                continue

            def pick(label):
                m = re.search(label + r"[：:]\s*([^\n]+)", found)
                return m.group(1).strip() if m else ""

            exec_money = pick("执行标的")
            amount = parse_amount(exec_money)

            details.append({
                "CaseCode": pick("案号") or case_code,
                "FilingDate": pick("立案时间"),
                "ExecCourt": pick("执行法院"),
                "ExecMoney": exec_money,
                "Amount": amount,
                "DetailStatus": "OK" if amount is not None else "NO_DETAIL_AMOUNT",
                "RawText": found
            })

        total = sum(d["Amount"] for d in details if d["Amount"] is not None)

        with open("execution-case-details.csv", "w", newline="", encoding="utf-8-sig") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "CaseCode",
                "FilingDate",
                "ExecCourt",
                "ExecMoney",
                "Amount",
                "DetailStatus",
                "RawText"
            ])
            writer.writeheader()
            writer.writerows(details)

        court_summary = defaultdict(lambda: {"Count": 0, "AmountSum": Decimal("0")})
        for d in details:
            if d["Amount"] is not None:
                court = d["ExecCourt"] or "UNKNOWN"
                court_summary[court]["Count"] += 1
                court_summary[court]["AmountSum"] += d["Amount"]

        with open("execution-amounts-by-court.csv", "w", newline="", encoding="utf-8-sig") as f:
            writer = csv.DictWriter(f, fieldnames=["ExecCourt", "Count", "AmountSum"])
            writer.writeheader()
            for court, item in sorted(
                court_summary.items(),
                key=lambda x: x[1]["AmountSum"],
                reverse=True
            ):
                writer.writerow({
                    "ExecCourt": court,
                    "Count": item["Count"],
                    "AmountSum": item["AmountSum"]
                })

        missing = [d["CaseCode"] for d in details if d["Amount"] is None]

        print("原始列表记录数:", len(all_rows))
        print("按案号去重案件数:", len(by_case))
        print("取到金额案件数:", sum(1 for d in details if d["Amount"] is not None))
        print("未取到金额案号:", missing)
        print("已取到金额案件合计:", total)

        browser.close()

if __name__ == "__main__":
    main()
```

## Required Final Report Format

When reporting to the user, include:

```text
查询日期：
查询主体：
查询范围：
原始列表记录数：
按案号去重案件数：
成功取到执行标的案件数：
未取到金额案号：
已取到金额案件合计：
```

If there are missing amounts, say:

```text
已取到金额的案件合计为 X；以下案号未取到公开执行标的：...
```

Do not say:

```text
全部案件总金额为 X
```

unless every unique `caseCode` has a valid `执行标的`.

## Recommended Output Files

Detail CSV:

```text
execution-case-details.csv
```

Columns:

```text
CaseCode
FilingDate
ExecCourt
ExecMoney
Amount
DetailStatus
RawText
```

Court summary CSV:

```text
execution-amounts-by-court.csv
```

Columns:

```text
ExecCourt
Count
AmountSum
```

## Failure Handling

### Blank Page

Cause:

```text
Headless mode or page challenge failure.
```

Fix:

```text
Use non-headless Chromium under xvfb-run.
Restart browser session.
Wait for #captchaImg.
```

### "网络繁忙"

Cause:

```text
JS challenge not passed.
```

Fix:

```text
Do not use pure HTTP.
Use a real browser and page context fetch.
```

### Captcha Looks Correct But Fails

Cause:

```text
Captcha image was re-requested and server-side captcha changed.
```

Fix:

```text
Screenshot #captchaImg from current page.
Use checkyzm to verify.
```

### Detail Page Returns 页面不存在

Cause:

```text
Wrong dePartyCardNum for that caseCode, or public detail unavailable.
```

Fix:

```text
Retry all dePartyCardNum values for the same caseCode.
If all fail, mark NO_DETAIL_AMOUNT.
```

### Main Page Loses State

Cause:

```text
Navigated main page to detailZhcx.do.
```

Fix:

```text
Use hidden iframe and set form.target = "detailFrame".
```

### Amount Total Is Incomplete

Cause:

```text
Some detail pages do not expose 执行标的.
```

Fix:

```text
Exclude null amounts from numeric sum.
Report missing caseCodes explicitly.
Use "已取到金额的案件合计".
```

## Field Semantics

`totalSize`:

```text
Raw list record count. May include duplicates.
```

`caseCode`:

```text
Case number. Use for final deduplicated case count.
```

`dePartyCardNum`:

```text
Encrypted value required by detail form. Keep all values under each caseCode.
```

`执行标的`:

```text
Execution amount from detail page. Use this for totals.
```

`DetailStatus`:

```text
OK = amount parsed
NO_DETAIL_AMOUNT = no retrievable public amount
```
