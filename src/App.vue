<script setup lang="ts">
import { ref, computed } from "vue";
import CsvInput from "./components/CsvInput.vue";
import JsonOutput from "./components/JsonOutput.vue";

// サンプルCSVデータ
const csvInput = ref(`id, user_name, age, is_active, score, note
101, "Tanaka, Taro", 28, true, 92.5, "Tokyo, Japan"
102, Suzuki Hanako, 34, false, 88.0, 
103, Sato Ken, 41, true, 76.5, "Line 1
Line 2"`);

const options = ref({
  sanitizeKeys: true,
  autoType: true,
  indent: 2,
});

function parseCSV(text: string): string[][] {
  const rows: string[][] = [];
  let currentRow: string[] = [];
  let currentVal = "";
  let inQuotes = false;
  let wasQuoted = false;

  const endField = () => {
    currentRow.push(wasQuoted ? currentVal : currentVal.trim());
    currentVal = "";
    wasQuoted = false;
  };
  const endRow = () => {
    endField();
    rows.push(currentRow);
    currentRow = [];
  };

  for (let i = 0; i < text.length; i++) {
    const char = text[i];
    const nextChar = text[i + 1];

    if (inQuotes) {
      if (char === '"' && nextChar === '"') {
        currentVal += '"';
        i++;
      } else if (char === '"') {
        inQuotes = false;
      } else {
        currentVal += char;
      }
    } else {
      if (char === '"' && !wasQuoted && currentVal.trim() === "") {
        currentVal = "";
        inQuotes = true;
        wasQuoted = true;
      } else if (char === ",") {
        endField();
      } else if (char === "\r" && nextChar === "\n") {
        endRow();
        i++;
      } else if (char === "\n" || char === "\r") {
        endRow();
      } else if (wasQuoted && /\s/.test(char)) {
        // 閉じ引用符の後ろの空白は無視する
      } else {
        currentVal += char;
      }
    }
  }

  if (inQuotes) {
    throw new Error('引用符(")が閉じられていません。');
  }

  if (currentVal !== "" || currentRow.length > 0 || wasQuoted) {
    endRow();
  }

  return rows.filter((row) => !(row.length === 1 && row[0].trim() === ""));
}

const JSON_NUMBER = /^-?(0|[1-9]\d*)(\.\d+)?([eE][+-]?\d+)?$/;

function isAmbiguousNumericKey(key: string): boolean {
  if (key === "") return false;
  const looksNumeric =
    !Number.isNaN(Number(key)) || /^[+-]?(NaN|Infinity)$/.test(key);
  return looksNumeric && !JSON_NUMBER.test(key);
}

function normalizeHeaders(rawHeaders: string[]) {
  const used = new Set<string>();
  const ambiguousKeys = new Set<string>();

  const headers = rawHeaders.map((raw, index) => {
    let key = raw.trim();

    if (options.value.sanitizeKeys) {
      key = key.replace(/[\u0000-\u001F\u007F]/g, "").replace(/\s+/g, "_");
      if (!key) key = `column_${index + 1}`;
    }

    if (isAmbiguousNumericKey(key)) ambiguousKeys.add(key);

    let unique = key;
    for (let n = 2; used.has(unique); n++) {
      unique = `${key}_${n}`;
    }
    used.add(unique);
    return unique;
  });

  return { headers, ambiguousKeys };
}

function parseValue(val: string): unknown {
  if (!options.value.autoType) return val;

  const trimmed = val.trim();
  if (trimmed === "") return null;
  if (trimmed.toLowerCase() === "true") return true;
  if (trimmed.toLowerCase() === "false") return false;
  if (trimmed.toLowerCase() === "null") return null;

  if (JSON_NUMBER.test(trimmed)) {
    const n = Number(trimmed);
    if (
      Number.isFinite(n) &&
      (!Number.isInteger(n) || Number.isSafeInteger(n))
    ) {
      return n;
    }
  }
  return val;
}

function stringifyRows(
  headers: string[],
  rows: unknown[][],
  indent: number,
): string {
  if (rows.length === 0) return "[]";

  const nl = indent ? "\n" : "";
  const sep = indent ? ": " : ":";
  const pad = (depth: number) => (indent ? " ".repeat(indent * depth) : "");

  const objects = rows.map((values) => {
    if (headers.length === 0) return `${pad(1)}{}`;
    const props = headers.map(
      (h, i) =>
        `${pad(2)}${JSON.stringify(h)}${sep}${JSON.stringify(values[i])}`,
    );
    return `${pad(1)}{${nl}${props.join("," + nl)}${nl}${pad(1)}}`;
  });

  return `[${nl}${objects.join("," + nl)}${nl}]`;
}

type ParsedResult = {
  jsonString: string;
  error: string | null;
  ambiguousKeys: Set<string>;
  mismatchRows: number[];
  columnCount: number;
};

const parsedResult = computed<ParsedResult>(() => {
  const empty: ParsedResult = {
    jsonString: "",
    error: null,
    ambiguousKeys: new Set<string>(),
    mismatchRows: [],
    columnCount: 0,
  };

  try {
    const rows = parseCSV(csvInput.value);
    if (rows.length === 0) return { ...empty, jsonString: "[]" };

    const { headers, ambiguousKeys } = normalizeHeaders(rows[0]);
    const mismatchRows: number[] = [];

    const values = rows.slice(1).map((row, i) => {
      if (row.length !== headers.length) mismatchRows.push(i + 2);
      return headers.map((_, idx) => parseValue(row[idx] ?? ""));
    });

    return {
      jsonString: stringifyRows(headers, values, options.value.indent),
      error: null,
      ambiguousKeys,
      mismatchRows,
      columnCount: headers.length,
    };
  } catch (e) {
    return {
      ...empty,
      error: e instanceof Error ? e.message : "パースエラーが発生しました",
    };
  }
});

const hasInvalidKeys = computed(
  () => parsedResult.value.ambiguousKeys.size > 0,
);

const mismatchMessage = computed(() => {
  const { mismatchRows, columnCount } = parsedResult.value;
  if (mismatchRows.length === 0) return "";
  const shown = mismatchRows.slice(0, 5).join(", ");
  const more = mismatchRows.length > 5 ? " ほか" : "";
  return `${mismatchRows.length}行で列数がヘッダー（${columnCount}列）と一致しません（行: ${shown}${more}）。不足分は null になります。`;
});

function decodeKeyToken(token: string): string | null {
  const raw = token
    .replace(/\s*:$/, "")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&amp;/g, "&");
  try {
    return JSON.parse(raw);
  } catch {
    return null;
  }
}

const highlightedJson = computed(() => {
  const json = parsedResult.value.jsonString;
  if (!json) return "";

  const escaped = json
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;");

  return escaped.replace(
    /("(\\u[a-zA-Z0-9]{4}|\\[^u]|[^\\"])*"(\s*:)?|\b(true|false|null)\b|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?)/g,
    (match) => {
      let cls = "json-number";
      if (/^"/.test(match)) {
        if (/:$/.test(match)) {
          const key = decodeKeyToken(match);
          cls =
            key !== null && parsedResult.value.ambiguousKeys.has(key)
              ? "json-invalid-key"
              : "json-key";
        } else {
          cls = "json-string";
        }
      } else if (/true|false/.test(match)) {
        cls = "json-boolean";
      } else if (/null/.test(match)) {
        cls = "json-null";
      }
      return `<span class="${cls}">${match}</span>`;
    },
  );
});
</script>

<template>
  <div class="csv-app-container">
    <!-- フローティング警告通知 -->
    <div v-if="hasInvalidKeys || mismatchMessage" class="toast-container">
      <div v-if="hasInvalidKeys" class="warning-toast">
        <span>⚠️</span>
        <span>
          数値と誤解されやすいキー（先頭ゼロ・16進数・NaN等）が含まれています。赤字部分をご確認ください。
        </span>
      </div>
      <div v-if="mismatchMessage" class="warning-toast">
        <span>⚠️</span>
        <span>{{ mismatchMessage }}</span>
      </div>
    </div>

    <div class="csv-app-card">
      <header class="header">
        <h2>CSV to JSON 変換くん</h2>
        <div class="controls">
          <label>
            <input type="checkbox" v-model="options.sanitizeKeys" />
            キーの正規化
          </label>
          <label>
            <input type="checkbox" v-model="options.autoType" />
            型自動変換
          </label>
          <label>
            インデント:
            <select v-model.number="options.indent">
              <option :value="2">2スペース</option>
              <option :value="4">4スペース</option>
              <option :value="0">圧縮</option>
            </select>
          </label>
        </div>
      </header>

      <div class="editor-grid">
        <!-- 入力ペイン -->
        <CsvInput v-model="csvInput" />

        <!-- 出力ペイン -->
        <JsonOutput
          :json-string="parsedResult.jsonString"
          :error="parsedResult.error"
          :highlighted-json="highlightedJson"
        />
      </div>
    </div>
  </div>
</template>
