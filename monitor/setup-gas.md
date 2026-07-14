# モニター募集フォームの保存先セットアップ（GAS → スプレッドシート）

回答データと計測イベントを、1つの Google スプレッドシートに自動で貯めるための設定です。**5分ほど**で終わります。

## 手順

1. Google スプレッドシートを新規作成（名前は「旬彩坊モニター_回答」など）
2. 上部メニュー **拡張機能 → Apps Script** を開く
3. 既存コードを全部消して、下の `コード.gs` を貼り付けて保存（💾）
4. 右上 **デプロイ → 新しいデプロイ** →
   - 種類：**ウェブアプリ**
   - 実行するユーザー：**自分**
   - アクセスできるユーザー：**全員**
   - 「デプロイ」→ 表示された **ウェブアプリURL**（`https://script.google.com/macros/s/…/exec`）をコピー
5. `monitor/index.html` の先頭スクリプトにある
   ```js
   var FORM_ENDPOINT = '';
   ```
   の `''` の中に、コピーしたURLを貼る →
   ```js
   var FORM_ENDPOINT = 'https://script.google.com/macros/s/AKfy……/exec';
   ```
6. 保存して push（GitHub Pages に反映）

> ⚠️ フォームの項目を増やしたときは、`toRow()` の列も合わせて増やしてください。未設定でもフォームは動きます（その場合はメール下書きが開く保険動作になります）。

---

## コード.gs（そのまま貼り付け）

```javascript
// 旬彩坊 店舗業務の見える化モニター：回答・計測の受け口
var SHEET_SUB = '回答';   // 申込データ
var SHEET_EV  = '計測';   // イベント（form_start / copy_intro など）

function doPost(e){
  try{
    var data = JSON.parse(e.postData.contents);
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    if(data.type === 'event'){
      writeRow_(ss, SHEET_EV,
        ['日時','イベント','流入元','店舗コード','役職','補足'],
        [nowJST_(), data.event||'', data.source||'', data.store||'', data.role||'', data.where||'']);
    } else {
      writeRow_(ss, SHEET_SUB,
        ['申込日時','状況','役職','店舗名','流入元','店舗コード','お名前','電話','メール','希望連絡','面談希望','面談メモ','同意','紹介者名','ギフト送付先','紹介元','自由記入'],
        [nowJST_(), data.status||'', data.role||'', data.store_name||'', data.source||'', data.store||'',
         data.name||'', data.tel||'', data.email||'', data.prefer||'', data.time||'', data.time_note||'', data.consent||'',
         data.ref_name||'', data.ref_contact||'', data.ref||'', data.note||'']);
    }
    return json_({ok:true});
  }catch(err){
    return json_({ok:false, error:String(err)});
  }
}

function doGet(){ return json_({ok:true, msg:'monitor endpoint alive'}); }

function writeRow_(ss, name, header, row){
  var sh = ss.getSheetByName(name);
  if(!sh){ sh = ss.insertSheet(name); sh.appendRow(header); }
  if(sh.getLastRow() === 0){ sh.appendRow(header); }
  sh.appendRow(row);
}
function nowJST_(){ return Utilities.formatDate(new Date(), 'Asia/Tokyo', 'yyyy-MM-dd HH:mm:ss'); }
function json_(o){ return ContentService.createTextOutput(JSON.stringify(o)).setMimeType(ContentService.MimeType.JSON); }
```

---

## 保存される内容

**「回答」シート**（1申込・1紹介登録 = 1行）
| 申込日時 | 状況 | 役職 | 店舗名 | 流入元 | 店舗コード | お名前 | 電話 | メール | 希望連絡 | 面談希望 | 面談メモ | 同意 | 紹介者名 | ギフト送付先 | 紹介元 | 自由記入 |

- **状況（status）** = `owner`（オーナー申込）／`referral`（スタッフの紹介登録）／`none`（興味なし）
- **紹介の突合**：スタッフが紹介を登録すると `status=referral`＋紹介者名・ギフト送付先の行が残る。そのスタッフが送ったURLからオーナーが申し込むと、オーナーの行（`status=owner`）の**「紹介元」列にスタッフ名が自動で入る**。→ 紹介者名＝紹介元 で突合し、面談完了後にギフト送付先へ発送。

**「計測」シート**（イベントごと）
| 日時 | イベント | 流入元 | 店舗コード | 役職 | 補足 |
- `form_start`（開始）／`referral_register`（紹介登録）／`copy_intro`（紹介文コピー）／`share_owner_url`（URL共有）／`demo_view`（デモ閲覧）／`meeting_apply`（面談申込）／`cta_click`（ボタン押下）など

## 流入元（source）の値
- `flyer` … 配送時の紙チラシ
- `tanom` … タノムのメッセージ
- `referral` … スタッフからオーナーへの共有リンク
- `direct` … パラメータなし直アクセス
