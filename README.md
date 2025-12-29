<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>월별·연령대별 내원수 자동 통계 (CSV 전용 · 브라우저 내 처리)</title>
  <style>
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Apple SD Gothic Neo,Noto Sans KR,sans-serif;margin:24px;line-height:1.45}
    h1{margin:0 0 8px 0}
    .muted{color:#6b7280;font-size:13px}
    .ok{color:#047857}
    .warn{color:#b45309}
    .danger{color:#b91c1c}
    .pill{display:inline-block;padding:2px 10px;border-radius:999px;background:#f3f4f6;font-size:12px;margin-left:8px}
    .card{border:1px solid #e5e7eb;border-radius:12px;padding:16px;margin:14px 0}
    .row{display:flex;gap:12px;flex-wrap:wrap;align-items:center}
    label{font-weight:700}
    select,input,button{padding:10px 12px;border-radius:10px;border:1px solid #d1d5db}
    button{cursor:pointer;font-weight:800}
    button:disabled{opacity:.5;cursor:not-allowed}
    .btnrow{display:flex;gap:10px;flex-wrap:wrap;align-items:center}
    table{border-collapse:collapse;width:100%;margin-top:12px}
    th,td{border:1px solid #e5e7eb;padding:8px 10px;text-align:right}
    th:first-child,td:first-child{text-align:left}
    th{background:#f9fafb}
    .note{background:#f9fafb;border:1px dashed #e5e7eb;border-radius:10px;padding:10px;margin-top:10px}
    .small{font-size:12px}
    .mono{font-family:ui-monospace,SFMono-Regular,Menlo,Monaco,Consolas,monospace}
  </style>
</head>
<body>
  <h1>월별·연령대별 내원수 자동 통계 <span class="pill">CSV 전용 · CDN 0개</span></h1>
  <p class="muted">업로드 파일은 서버로 전송되지 않고, <b>사용자 브라우저 안에서만</b> 계산됩니다.</p>

  <div class="card">
    <div class="row">
      <label>CSV 업로드</label>
      <input id="file" type="file" accept=".csv" />
      <span id="fileInfo" class="muted"></span>
    </div>

    <div class="row" style="margin-top:12px">
      <label>내원일(날짜) 열</label>
      <select id="dateCol" disabled></select>

      <label>주민번호 열</label>
      <select id="rrnCol" disabled></select>

      <label>열문자 빠른선택</label>
      <input id="dateLetter" class="small" placeholder="내원일 예: P" style="width:140px" disabled />
      <input id="rrnLetter"  class="small" placeholder="주민번호 예: E" style="width:140px" disabled />
      <button id="applyLetters" disabled>적용</button>
    </div>

    <div class="row" style="margin-top:12px">
      <label>연도</label>
      <select id="yearFilter" disabled></select>

      <div class="btnrow">
        <button id="run" disabled>통계 생성</button>
        <button id="download" disabled>결과 CSV 다운로드</button>
      </div>
    </div>

    <div class="row" style="margin-top:10px">
      <div class="btnrow">
        <button id="showMissing" disabled>날짜미기록 상세보기</button>
        <button id="downloadMissing" disabled>날짜미기록 CSV 다운로드</button>
        <button id="showInvalid" disabled>날짜해석불가 상세보기</button>
        <button id="downloadInvalid" disabled>날짜해석불가 CSV 다운로드</button>
      </div>
    </div>

    <p class="muted" id="encodingInfo" style="margin-top:10px"></p>
    <p class="muted" id="delimiterInfo" style="margin-top:6px"></p>
    <p class="muted warn" id="hint" style="margin-top:6px"></p>

    <div class="note muted">
      <b>자동 날짜 처리 규칙</b><br>
      - 숫자만 추출해 처리(공백/기호 섞여도 OK)<br>
      - 8자리: <span class="mono">YYYYMMDD</span><br>
      - 10자리: <span class="mono">YYYYMMDDHH</span> → 분을 <span class="mono">00</span>으로 자동보정<br>
      - 12자리: <span class="mono">YYYYMMDDHHMM</span><br>
      - 14자리: <span class="mono">YYYYMMDDHHMMSS</span><br>
      <br>
      <b>날짜 오류 분리</b><br>
      - <b>날짜미기록</b>: 빈칸/숫자 없음(예약/미내원 등 포함) → 통계에서 제외(결측치로 분리)<br>
      - <b>날짜해석불가</b>: 숫자는 있는데 위 규칙으로 날짜 생성 실패 → 통계에서 제외(진짜 오류)
    </div>
  </div>

  <div class="card">
    <h3 style="margin:0 0 8px 0">결과</h3>
    <div id="result" class="muted">CSV 파일을 업로드한 뒤, 열을 선택하고 통계 생성을 누르세요.</div>
  </div>

<script>
  const $ = (id) => document.getElementById(id);

  const fileInput = $("file");
  const fileInfo   = $("fileInfo");
  const dateColSel = $("dateCol");
  const rrnColSel  = $("rrnCol");
  const yearSel    = $("yearFilter");

  const runBtn     = $("run");
  const downBtn    = $("download");

  const showMissingBtn = $("showMissing");
  const downMissingBtn = $("downloadMissing");
  const showInvalidBtn = $("showInvalid");
  const downInvalidBtn = $("downloadInvalid");

  const resultDiv  = $("result");
  const hint       = $("hint");
  const encodingInfo = $("encodingInfo");
  const delimiterInfo = $("delimiterInfo");

  const dateLetter = $("dateLetter");
  const rrnLetter  = $("rrnLetter");
  const applyLetters = $("applyLetters");

  let rows = [];
  let headers = [];
  let lastTable = null;

  let missingDateRows = []; // 날짜미기록
  let invalidDateRows = []; // 날짜해석불가

  // ✅ 1~12월 전체
  const MONTHS = [1,2,3,4,5,6,7,8,9,10,11,12];

  // ✅ 요청하신 연령대 구간
  const AGE_BANDS = [
    {key:"20대미만",  test:(age)=>age < 20},
    {key:"20대",      test:(age)=>age >=20 && age < 30},
    {key:"30대",      test:(age)=>age >=30 && age < 40},
    {key:"40대",      test:(age)=>age >=40 && age < 50},
    {key:"50~64세",   test:(age)=>age >=50 && age < 65},
    {key:"65세이상",  test:(age)=>age >=65},
  ];

  // ---------------- 인코딩 자동 감지 ----------------
  function decodeWith(encoding, arrayBuffer){
    const dec = new TextDecoder(encoding, {fatal:false});
    return dec.decode(arrayBuffer);
  }
  function hasManyNullBytes(uint8){
    let nulls = 0;
    const n = Math.min(uint8.length, 4000);
    for(let i=0;i<n;i++) if(uint8[i] === 0) nulls++;
    return (n>0) && (nulls / n) > 0.1;
  }
  function detectCsvEncoding(arrayBuffer){
    const u8 = new Uint8Array(arrayBuffer);

    // BOM
    if(u8.length >= 2){
      if(u8[0] === 0xFF && u8[1] === 0xFE){
        return {encoding:"UTF-16LE(BOM)", text: decodeWith("utf-16le", arrayBuffer)};
      }
      if(u8[0] === 0xFE && u8[1] === 0xFF){
        // UTF-16BE -> swap -> decode as UTF-16LE
        const swapped = new Uint8Array(u8.length);
        for(let i=0;i<u8.length-1;i+=2){ swapped[i] = u8[i+1]; swapped[i+1] = u8[i]; }
        return {encoding:"UTF-16BE(BOM)", text: decodeWith("utf-16le", swapped.buffer)};
      }
    }
    if(u8.length >= 3 && u8[0]===0xEF && u8[1]===0xBB && u8[2]===0xBF){
      return {encoding:"UTF-8(BOM)", text: decodeWith("utf-8", arrayBuffer)};
    }

    // UTF-16 추정
    if(hasManyNullBytes(u8)){
      return {encoding:"UTF-16LE(추정)", text: decodeWith("utf-16le", arrayBuffer)};
    }

    // UTF-8 시도
    const t8 = decodeWith("utf-8", arrayBuffer);
    const rep = (t8.match(/\uFFFD/g) || []).length;
    if(t8.length > 0 && (rep / t8.length) <= 0.001){
      return {encoding:"UTF-8", text: t8};
    }

    // CP949(EUC-KR)
    return {encoding:"CP949(EUC-KR)", text: decodeWith("euc-kr", arrayBuffer)};
  }

  // ---------------- 구분자 자동 감지/파싱 ----------------
  function detectDelimiter(firstLine){
    firstLine = firstLine.replace(/^\uFEFF/, "");
    if(firstLine.includes("\t")) return "\t";
    const candidates = [",",";","|"];
    let best = {delim:",", count:0};
    for(const d of candidates){
      const c = firstLine.split(d).length - 1;
      if(c > best.count) best = {delim:d, count:c};
    }
    return (best.count === 0) ? "\t" : best.delim;
  }

  function parseDelimited(text){
    text = text.replace(/\r\n/g,"\n").replace(/\r/g,"\n");
    const firstLine = text.split("\n")[0] ?? "";
    const delim = detectDelimiter(firstLine);

    const out = [];
    let row = [];
    let cur = "";
    let inQuotes = false;

    for(let i=0;i<text.length;i++){
      const ch = text[i];

      if(inQuotes){
        if(ch === '"'){
          if(text[i+1] === '"'){ cur += '"'; i++; }
          else inQuotes = false;
        } else cur += ch;
      } else {
        if(ch === '"') inQuotes = true;
        else if(ch === delim){ row.push(cur); cur=""; }
        else if(ch === "\n"){ row.push(cur); out.push(row); row=[]; cur=""; }
        else cur += ch;
      }
    }
    row.push(cur); out.push(row);

    while(out.length && out[out.length-1].every(v => String(v).trim()==="")) out.pop();
    return {grid: out, delim};
  }

  function fillSelectOptions(sel, opts, placeholder){
    sel.innerHTML = "";
    const ph = document.createElement("option");
    ph.value = "";
    ph.textContent = placeholder;
    sel.appendChild(ph);
    for(const o of opts){
      const op = document.createElement("option");
      op.value = o;
      op.textContent = o;
      sel.appendChild(op);
    }
  }

  function guessColumn(headers, patterns){
    const lower = headers.map(h => String(h).toLowerCase());
    for(let i=0;i<lower.length;i++){
      for(const p of patterns){
        if(lower[i].includes(p)) return headers[i];
      }
    }
    return "";
  }

  // ---------------- 날짜 파서 (8/10/12/14 + 일반포맷) ----------------
  function parseDate(val){
    if(val == null || val === "") return null;

    if(typeof val === "number" && isFinite(val)){
      val = Math.round(val);
    }

    let raw = String(val).trim();
    if(!raw) return null;

    // 과학표기(e+) 처리
    if(/[eE]\+?\d+/.test(raw)){
      const n = Number(raw);
      if(isFinite(n)) raw = String(Math.round(n));
    }

    const digits = raw.replace(/[^0-9]/g, "");
    if(!digits) return null;

    // 8자리 YYYYMMDD
    if(digits.length === 8){
      const y = +digits.slice(0,4), mo = +digits.slice(4,6), da = +digits.slice(6,8);
      const d = new Date(y, mo-1, da);
      return isNaN(d) ? null : d;
    }

    // 10자리 YYYYMMDDHH -> HH:00 보정
    if(digits.length === 10){
      const y = +digits.slice(0,4), mo = +digits.slice(4,6), da = +digits.slice(6,8);
      const hh = +digits.slice(8,10);
      const d = new Date(y, mo-1, da, hh, 0, 0, 0);
      return isNaN(d) ? null : d;
    }

    // 12자리 YYYYMMDDHHMM
    if(digits.length === 12){
      const y = +digits.slice(0,4), mo = +digits.slice(4,6), da = +digits.slice(6,8);
      const hh = +digits.slice(8,10), mm = +digits.slice(10,12);
      const d = new Date(y, mo-1, da, hh, mm, 0, 0);
      return isNaN(d) ? null : d;
    }

    // 14자리 YYYYMMDDHHMMSS
    if(digits.length === 14){
      const y = +digits.slice(0,4), mo = +digits.slice(4,6), da = +digits.slice(6,8);
      const hh = +digits.slice(8,10), mm = +digits.slice(10,12), ss = +digits.slice(12,14);
      const d = new Date(y, mo-1, da, hh, mm, ss, 0);
      return isNaN(d) ? null : d;
    }

    // 일반 포맷 (YYYY-MM-DD 등)
    const t = String(val).trim();
    let m = t.match(/^(\d{4})\s*[-/.]\s*(\d{1,2})\s*[-/.]\s*(\d{1,2})/);
    if(m){
      const y=+m[1], mo=+m[2], da=+m[3];
      const d = new Date(y, mo-1, da);
      return isNaN(d) ? null : d;
    }

    const d2 = new Date(t);
    return isNaN(d2) ? null : d2;
  }

  // ---------------- 주민번호 -> 생년월일 ----------------
  function normalizeRRN(val){
    if(val == null) return "";
    return String(val).replace(/\s/g,"").replace(/[^0-9]/g,"");
  }

  function parseBirthFromRRN(rrn){
    const s = normalizeRRN(rrn);
    if(s.length < 7) return null;

    const yy = parseInt(s.slice(0,2),10);
    const mm = parseInt(s.slice(2,4),10);
    const dd = parseInt(s.slice(4,6),10);
    const g  = parseInt(s.slice(6,7),10);

    if(!(mm>=1 && mm<=12 && dd>=1 && dd<=31)) return null;

    let century = null;
    if([1,2,5,6].includes(g)) century = 1900;
    else if([3,4,7,8].includes(g)) century = 2000;
    else if([9,0].includes(g)) century = 1800;
    else return null;

    const yyyy = century + yy;
    const d = new Date(yyyy, mm-1, dd);
    if(d.getFullYear() !== yyyy || d.getMonth() !== (mm-1) || d.getDate() !== dd) return null;
    return d;
  }

  function ageAt(birth, onDate){
    let age = onDate.getFullYear() - birth.getFullYear();
    const m = onDate.getMonth() - birth.getMonth();
    if(m < 0 || (m === 0 && onDate.getDate() < birth.getDate())) age--;
    return age;
  }

  function bandOf(age){
    if(age == null || isNaN(age) || age < 0 || age > 130) return null;
    for(const b of AGE_BANDS) if(b.test(age)) return b.key;
    return null;
  }

  // ---------------- 연도 목록: 선택된 내원일 열 기준으로 항상 새로 계산 ----------------
  function detectYearOptions(dateCol){
    const years = new Set();
    for(const r of rows){
      const d = parseDate(r[dateCol]);
      if(d) years.add(d.getFullYear());
    }
    return Array.from(years).sort((a,b)=>b-a); // 최신연도부터
  }

  function refreshYearOptions(){
    const dateCol = dateColSel.value;
    if(!dateCol){
      fillSelectOptions(yearSel, [], "전체(연도무관)");
      yearSel.value = "";
      return;
    }
    const years = detectYearOptions(dateCol);
    fillSelectOptions(yearSel, years.map(String), "전체(연도무관)");
    yearSel.value = ""; // 기본값: 전체
  }

  // ---------------- 통계 생성 ----------------
  function buildStats(dateCol, rrnCol, yearFilter){
    missingDateRows = [];
    invalidDateRows = [];

    const counts = {};
    for(const b of AGE_BANDS){
      counts[b.key] = Object.fromEntries(MONTHS.map(m => [m,0]));
    }

    let missingDate = 0; // 빈칸/숫자 없음
    let badDate = 0;     // 숫자는 있는데 파싱 실패
    let badRRN = 0;
    let outMonth = 0;
    let outYear = 0;

    for(const r of rows){
      const rawDate = r[dateCol];
      const rawStr = (rawDate == null) ? "" : String(rawDate).trim();
      const digitOnly = rawStr.replace(/[^0-9]/g,"");

      // ✅ 날짜 미기록(결측치): 빈칸/숫자 없음(예약/미내원 등)
      if(!rawStr || digitOnly.length === 0){
        missingDate++;
        missingDateRows.push({ 원본내원일: rawDate, 주민번호: r[rrnCol] });
        continue;
      }

      const d = parseDate(rawDate);

      // ✅ 숫자는 있는데 날짜로 못 만듦 = 진짜 해석불가
      if(!d){
        badDate++;
        invalidDateRows.push({ 원본내원일: rawDate, 주민번호: r[rrnCol] });
        continue;
      }

      const y = d.getFullYear();
      if(yearFilter && y !== yearFilter){ outYear++; continue; }

      const month = d.getMonth() + 1;
      if(!MONTHS.includes(month)){ outMonth++; continue; }

      const birth = parseBirthFromRRN(r[rrnCol]);
      if(!birth){ badRRN++; continue; }

      const age = ageAt(birth, d);
      const band = bandOf(age);
      if(!band){ badRRN++; continue; }

      counts[band][month] += 1;
    }

    return { counts, meta: { missingDate, badDate, badRRN, outMonth, outYear } };
  }

  function matrixFromCounts(counts){
    const cols = MONTHS.slice();
    const rowsOut = AGE_BANDS.map(b => ({
      label: b.key,
      values: cols.map(m => counts[b.key][m] || 0)
    }));
    return { cols, rows: rowsOut };
  }

  function renderTable(matrix){
    const {cols, rows:rr} = matrix;
    let html = "<table><thead><tr><th>연령대</th>";
    for(const c of cols) html += `<th>${c}월</th>`;
    html += `<th>합계</th></tr></thead><tbody>`;

    for(const row of rr){
      html += `<tr><td>${row.label}</td>`;
      let sum = 0;
      for(const v of row.values){ sum += v; html += `<td>${v.toLocaleString()}</td>`; }
      html += `<td><b>${sum.toLocaleString()}</b></td></tr>`;
    }

    html += `<tr><td><b>합계</b></td>`;
    let grand = 0;
    for(let i=0;i<cols.length;i++){
      let colSum = 0;
      for(const row of rr) colSum += row.values[i];
      grand += colSum;
      html += `<td><b>${colSum.toLocaleString()}</b></td>`;
    }
    html += `<td><b>${grand.toLocaleString()}</b></td></tr>`;
    html += "</tbody></table>";
    return html;
  }

  // ---------------- 다운로드 helpers ----------------
  function toCsvCell(v){
    const s = (v == null) ? "" : String(v);
    return /[",\n]/.test(s) ? `"${s.replace(/"/g,'""')}"` : s;
  }

  function downloadCSV(filename, rows2d){
    const csv = rows2d.map(r => r.map(toCsvCell).join(",")).join("\n");
    const blob = new Blob([csv], {type:"text/csv;charset=utf-8"});
    const a = document.createElement("a");
    a.download = filename;
    a.href = URL.createObjectURL(blob);
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(a.href);
  }

  function downloadResultCSV(matrix, yearVal){
    const header = ["연령대", ...matrix.cols.map(m => `${m}월`), "합계"];
    const lines = [header];

    for(const r of matrix.rows){
      const sum = r.values.reduce((a,b)=>a+b,0);
      lines.push([r.label, ...r.values, sum]);
    }

    const colTotals = matrix.cols.map((m, idx) => matrix.rows.reduce((s,r)=>s+r.values[idx],0));
    const grand = colTotals.reduce((a,b)=>a+b,0);
    lines.push(["합계", ...colTotals, grand]);

    const y = yearVal ? String(yearVal) : "전체";
    downloadCSV(`월별_연령대별_내원수_1-12월_${y}.csv`, lines);
  }

  function maskRRNKeepLast2(rrn){
    const s = (rrn == null) ? "" : String(rrn);
    if(!s) return "";
    const last2 = s.slice(-2);
    const masked = s.slice(0, Math.max(0, s.length-2)).replace(/[0-9]/g,"*") + last2;
    return masked;
  }

  function downloadIssueCSV(kind, list){
    const lines = [["원본내원일","주민번호(마스킹)","주민번호_마지막2자리"]];
    for(const r of list){
      const rrn = (r.주민번호 == null) ? "" : String(r.주민번호);
      lines.push([r.원본내원일, maskRRNKeepLast2(rrn), rrn.slice(-2)]);
    }
    downloadCSV(`${kind}_${list.length}건.csv`, lines);
  }

  function renderIssueTable(kindTitle, list){
    if(!list || list.length === 0){
      resultDiv.innerHTML = `<div class="ok"><b>🎉 ${kindTitle} (0건)</b></div>`;
      return;
    }
    const head = `<div class="muted"><b class="danger">${kindTitle} ${list.length}건</b> (주민번호는 마지막 2자리만 표시)</div>`;
    let html = head + `<table><thead><tr><th>번호</th><th>원본 내원일</th><th>주민번호(끝2)</th></tr></thead><tbody>`;
    const maxShow = 300;
    list.slice(0, maxShow).forEach((r, i) => {
      const rrn = (r.주민번호 == null) ? "" : String(r.주민번호);
      html += `<tr>
        <td>${i+1}</td>
        <td style="text-align:left">${r.원본내원일 == null ? "" : String(r.원본내원일)}</td>
        <td>${rrn.slice(-2)}</td>
      </tr>`;
    });
    html += `</tbody></table>`;
    if(list.length > maxShow){
      html += `<div class="muted">※ 화면에는 ${maxShow}건까지만 표시됩니다. 전체는 CSV 다운로드를 사용하세요.</div>`;
    }
    resultDiv.innerHTML = html;
  }

  // ---------------- 열문자 매핑 ----------------
  function colLetterToIndex(letter){
    const s = String(letter||"").trim().toUpperCase();
    if(!s) return null;
    let n = 0;
    for(let i=0;i<s.length;i++){
      const c = s.charCodeAt(i);
      if(c < 65 || c > 90) return null;
      n = n*26 + (c - 64);
    }
    return n - 1; // A=0
  }

  function applyLetterMapping(){
    const di = colLetterToIndex(dateLetter.value);
    const ri = colLetterToIndex(rrnLetter.value);
    if(di == null || ri == null){
      hint.textContent = "열문자는 A~Z(또는 AA, AB...) 형태로 입력해주세요. 예: P, E";
      return;
    }
    if(di >= headers.length || ri >= headers.length){
      hint.textContent = `열문자 범위 오류: 현재 컬럼 수는 ${headers.length}개 입니다.`;
      return;
    }
    dateColSel.value = headers[di];
    rrnColSel.value  = headers[ri];
    hint.textContent = `열문자 적용 완료: 내원일=${dateLetter.value.toUpperCase()} / 주민번호=${rrnLetter.value.toUpperCase()}`;
    refreshYearOptions(); // ✅ 내원일 열이 바뀌면 연도도 즉시 갱신
  }

  applyLetters.addEventListener("click", applyLetterMapping);

  // ✅ 내원일 열을 사용자가 바꾸면 연도 목록 자동 갱신 (연도 안 뜨는 문제 해결)
  dateColSel.addEventListener("change", refreshYearOptions);

  // ---------------- 업로드 처리 ----------------
  fileInput.addEventListener("change", async (e) => {
    hint.textContent = "";
    encodingInfo.textContent = "";
    delimiterInfo.textContent = "";
    resultDiv.textContent = "읽는 중…";

    lastTable = null;
    missingDateRows = [];
    invalidDateRows = [];

    // 버튼 비활성화
    runBtn.disabled = true;
    downBtn.disabled = true;
    showMissingBtn.disabled = true;
    downMissingBtn.disabled = true;
    showInvalidBtn.disabled = true;
    downInvalidBtn.disabled = true;

    dateColSel.disabled = true;
    rrnColSel.disabled = true;
    yearSel.disabled = true;
    dateLetter.disabled = true;
    rrnLetter.disabled = true;
    applyLetters.disabled = true;

    const f = e.target.files?.[0];
    if(!f){
      resultDiv.textContent = "CSV 파일을 업로드하면 열 선택 후 통계를 생성할 수 있어요.";
      return;
    }
    fileInfo.textContent = `${f.name} (${Math.round(f.size/1024).toLocaleString()} KB)`;

    try{
      const buf = await f.arrayBuffer();
      const {encoding, text} = detectCsvEncoding(buf);
      encodingInfo.innerHTML = `CSV 인코딩 자동 감지: <b class="ok">${encoding}</b>`;

      const parsed = parseDelimited(text);
      const grid = parsed.grid;

      delimiterInfo.innerHTML = `구분자 자동 감지: <b class="ok">${parsed.delim === "\t" ? "탭(\\t)" : parsed.delim}</b>`;

      if(grid.length < 2){
        resultDiv.textContent = "CSV에 데이터가 충분하지 않습니다.";
        return;
      }

      headers = grid[0].map(h => String(h).trim());
      const body = grid.slice(1);

      rows = body.map(cols => {
        const obj = {};
        for(let i=0;i<headers.length;i++){
          obj[headers[i]] = cols[i] ?? "";
        }
        return obj;
      });

      fillSelectOptions(dateColSel, headers, "선택");
      fillSelectOptions(rrnColSel, headers, "선택");

      // 자동 컬럼 추정
      const guessDate = guessColumn(headers, ["내원", "진료", "date", "visit", "접수", "일자"]);
      const guessRRN  = guessColumn(headers, ["주민", "rrn", "주민번호", "id", "생년", "주민등록"]);
      if(guessDate) dateColSel.value = guessDate;
      if(guessRRN)  rrnColSel.value  = guessRRN;

      // ✅ 연도 목록은 선택된 내원일 열 기준으로 생성
      refreshYearOptions();

      // UI 활성화
      dateColSel.disabled = false;
      rrnColSel.disabled = false;
      yearSel.disabled = false;
      runBtn.disabled = false;

      dateLetter.disabled = false;
      rrnLetter.disabled = false;
      applyLetters.disabled = false;

      resultDiv.innerHTML = `<div class="muted">
        열을 선택(또는 열문자 E/P 입력 후 적용)한 뒤 <b>통계 생성</b>을 눌러주세요.
        (총 ${rows.length.toLocaleString()}행)
      </div>`;
      hint.textContent = "주민번호는 숫자만 추출해 계산합니다(‘-’ 포함 가능). 내원일은 숫자형 날짜(YYYYMMDDHHMM 등)를 자동 처리합니다.";
    }catch(err){
      console.error(err);
      resultDiv.textContent = "파일을 읽는 중 오류가 발생했어요. CSV가 손상되었거나 포맷이 특이할 수 있습니다.";
      hint.textContent = "CSV 첫 줄(컬럼명 라인) 또는 내원일 값 예시 몇 개를 주시면 포맷에 맞춰 수정해드릴게요.";
    }
  });

  // ---------------- 통계 생성 버튼 ----------------
  runBtn.addEventListener("click", () => {
    const dateCol = dateColSel.value;
    const rrnCol = rrnColSel.value;
    const yearVal = yearSel.value ? parseInt(yearSel.value,10) : null;

    if(!dateCol || !rrnCol){
      resultDiv.textContent = "내원일 열과 주민번호 열을 선택해주세요.";
      return;
    }

    const {counts, meta} = buildStats(dateCol, rrnCol, yearVal);
    const matrix = matrixFromCounts(counts);

    resultDiv.innerHTML = `
      <div class="muted">
        기준: <b>${yearVal ? yearVal+"년" : "전체 연도"}</b> / <b>1~12월</b> / 나이=내원일 기준(만 나이)
        <br>제외:
        날짜미기록 <b class="warn">${meta.missingDate.toLocaleString()}</b>건,
        날짜해석불가 <b class="danger">${meta.badDate.toLocaleString()}</b>건,
        주민번호해석불가 ${meta.badRRN.toLocaleString()}건,
        1~12월 외 ${meta.outMonth.toLocaleString()}건
        ${yearVal ? `, 연도불일치 ${meta.outYear.toLocaleString()}건` : ""}.
      </div>
      ${renderTable(matrix)}
    `;

    lastTable = {matrix, yearVal};

    // 버튼 활성화
    downBtn.disabled = false;

    showMissingBtn.disabled = false;
    downMissingBtn.disabled = (missingDateRows.length === 0);

    showInvalidBtn.disabled = false;
    downInvalidBtn.disabled = (invalidDateRows.length === 0);
  });

  // ---------------- 결과 CSV 다운로드 ----------------
  downBtn.addEventListener("click", () => {
    if(!lastTable) return;
    downloadResultCSV(lastTable.matrix, lastTable.yearVal);
  });

  // ---------------- 상세보기/다운로드: 날짜미기록 ----------------
  showMissingBtn.addEventListener("click", () => {
    renderIssueTable("날짜미기록", missingDateRows);
  });
  downMissingBtn.addEventListener("click", () => {
    if(!missingDateRows || missingDateRows.length === 0){
      alert("날짜미기록 건이 없습니다(0건).");
      return;
    }
    downloadIssueCSV("날짜미기록", missingDateRows);
  });

  // ---------------- 상세보기/다운로드: 날짜해석불가 ----------------
  showInvalidBtn.addEventListener("click", () => {
    renderIssueTable("날짜해석불가", invalidDateRows);
  });
  downInvalidBtn.addEventListener("click", () => {
    if(!invalidDateRows || invalidDateRows.length === 0){
      alert("날짜해석불가 건이 없습니다(0건).");
      return;
    }
    downloadIssueCSV("날짜해석불가", invalidDateRows);
  });
</script>
</body>
</html>
