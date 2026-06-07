# 救急隊観察シミュレーター Ver.3 症例追加ガイド

すべての症例データは `index_ver3.html` 内に直接記述されています。

## 追加先

- **固定症例**（カードに常時表示したい場合）: `index_ver3.html` の `SCENARIOS` 配列に追加します。
- **症例ガチャ**（ランダム生成で出題したい場合）: `index_ver3.html` の `TEMPLATES` 配列に追加します。

固定症例とガチャ症例では構造が少し異なります（下記）。

## 固定症例（SCENARIOS）の構造

各項目は固定値で記述します。

- `id`: 英数字の識別子
- `title`: 症例名
- `level`: 難易度や重症度の表示文
- `diff`: `mid` または `hard`
- `brief`: 指令内容
- `demo`: 第一印象
- `truth`: 正解の疾患・重症度・搬送先 `{dx, sev, dest}`
- `vit0`: 初期バイタル
- `model`: 時間経過によるバイタル変化
- `tx`: 処置（属性は下記）
- `events`: 時間経過イベント
- `history`: SAMPLE/OPQRST
- `family`: 家族・関係者聴取
- `exam`: 全身観察
- `dxOptions`: 疾患推定の選択肢
- `destOptions`: 搬送先選択肢
- `aiQ`: デブリーフィング質問
- `driftScale`（任意）: 悪化速度の固定（例 `0.2`）
- `learning`（任意）: 鑑別・病態解説の上書き（下記「学習解説」参照）

## 症例ガチャ（TEMPLATES）の構造

ガチャ症例は年齢・性別・初期バイタルをランダム化するため、`title` / `brief` / `demo` / `vit0` などを**関数**で記述します。

- `id` / `diff` / `level` / `truth` / `model` / `tx` / `events` / `history` / `family` / `exam` / `dxOptions` / `destOptions` / `aiQ`: 固定症例と同じ
- `cat`: カテゴリ。`"内因"` / `"外傷"` / `"小児・産科"`（ガチャの絞り込みに使用）
- `tier`: レベル。`"初級"` / `"上級"`（ガチャの絞り込みに使用）
- `person`: 患者属性を生成する関数。`()=>PERS(最小年齢, 最大年齢, 性別省略可)`
- `title` / `brief` / `demo`: 患者属性 `p`（`p.age`, `p.sex`）を受け取る関数
- `vit0`: 初期バイタルを生成する関数（`R(min,max)` 整数乱数、`R1(min,max)` 小数、`dbpOf(sbp)`、`jcsOf(gcs)` などのヘルパーを使用）
- `airwayNote`（任意）: 気道所見の補足

## 採点で重要な指定

- 重症例は `diff:"hard"`、または `level` / `truth.sev` に「重症」を含めます。
- 早期処置評価を強くしたい処置には `heal:true` を付けます。
- 必須級の処置には `key:true` を付けます。
- 法令・医学的に避けるべき処置には `wrong:true` を付けます（実際に減点されます）。
- 根拠確認後でないと実施できない処置には `requires:"flag名"` を付け、前段処置に `reveal:"flag名"` を付けます（例: 低血糖症例の `"hypoConfirmed"`）。
- バイタル悪化は開始から一定時間（重症例は早め）まで緩徐です。その後、`heal:true` の処置や重要処置の多くが未実施だと悪化速度が上がります。
- 症例ごとに悪化速度を固定したい場合は `driftScale:0.2` のように指定できます。

### 処置（tx）の属性まとめ

```js
"処置名":{ spo2:+8, rr:-3, note:"効果の説明",
           key:true,        // 重要処置（採点対象）
           heal:true,       // 救命的・早期実施を評価
           wrong:true,      // 不適切処置（減点）
           requires:"flag", // このflagが立つまで実施不可
           reveal:"flag" }  // 実施するとflagを立てる
```

## 学習解説（鑑別・病態）

終了後の「鑑別診断と除外根拠」「病態変化の解説」は、`index_ver3.html` の `learningProfile()` 関数で表示されます。
疾患の判定は `id` / `truth.dx` / `title` の文字列一致で自動的に行われ、既存16疾患には解説が用意されています。

新しい疾患を追加して独自の解説を出したい場合は、症例オブジェクトに `learning` を直接指定すると `learningProfile()` の自動判定より優先されます。

```js
learning:{
  rationaleOptions:[
    {label:"冷汗・蒼白", key:true, hint:"循環不全を示唆"}
  ],
  differentials:[
    {dx:"鑑別疾患名", whyConsider:"なぜ鑑別に挙がるか", whyLessLikely:"なぜ考えにくいか"}
  ],
  pathophys:{ deterioration:"悪化の理由", improvement:"処置で改善する理由" }
}
```

## 最小サンプル（固定症例）

```js
{
  id:"sample",
  title:"息苦しさを訴える70代男性",
  level:"重症",
  diff:"hard",
  brief:"指令：70代男性、自宅で呼吸困難。",
  demo:"70代 男性 / 起座位、会話困難。",
  truth:{dx:"急性心不全(肺水腫)",sev:"重症",dest:"三次(循環器対応・PCI可能)"},
  vit0:{hr:120,sbp:180,dbp:112,spo2:86,rr:32,bt:36.6,gcs:14,jcs:"1",pupil:"3.0mm 両側 対光反射あり",skin:"冷汗・湿潤"},
  model:{hr:{drift:0.2,floor:90,ceil:150},sbp:{drift:-0.2,floor:120,ceil:210},spo2:{drift:-0.35,floor:72,ceil:99},rr:{drift:0.3,floor:16,ceil:42}},
  tx:{
    "高流量酸素投与":{spo2:9,note:"SpO2上昇",key:true,heal:true},
    "起座位":{rr:-3,spo2:2,note:"呼吸仕事量が軽減",key:true}
  },
  events:[],
  history:{"S 主訴・症状":"息苦しい。","A アレルギー":"なし。"},
  family:{"家族へ：いつから？":"夜間から急に悪化しました。"},
  exam:{"胸部":"両下肺に湿性ラ音。","四肢":"下腿浮腫あり。"},
  dxOptions:["急性心不全(肺水腫)","気管支喘息発作","肺炎"],
  destOptions:["三次(循環器対応・PCI可能)","二次(一般救急)"],
  aiQ:["心不全を疑った根拠を振り返ってください。"]
}
```

## 最小サンプル（症例ガチャ）

```js
{
  id:"sample_g", cat:"内因", tier:"上級", diff:"hard", level:"重症",
  truth:{dx:"急性心不全(肺水腫)",sev:"重症",dest:"三次(循環器対応・PCI可能)"},
  person:()=>PERS(60,85,"男性"),
  title:p=>`息苦しさを訴える${p.age}歳${p.sex}`,
  brief:p=>`指令：${p.age}歳${p.sex}、自宅で呼吸困難。`,
  demo:p=>`${p.age}歳 ${p.sex} / 起座位、会話困難。`,
  vit0:()=>{const sbp=R(160,200);return{hr:R(110,130),sbp,dbp:dbpOf(sbp),spo2:R(82,90),rr:R(28,36),bt:R1(36.2,36.8),gcs:14,jcs:jcsOf(14),pupil:"3.0mm 両側 対光反射あり",skin:"冷汗・湿潤"};},
  model:{hr:{drift:0.2,floor:90,ceil:150},sbp:{drift:-0.2,floor:120,ceil:210},spo2:{drift:-0.35,floor:72,ceil:99},rr:{drift:0.3,floor:16,ceil:42}},
  tx:{"高流量酸素投与":{spo2:9,note:"SpO2上昇",key:true,heal:true},"起座位":{rr:-3,spo2:2,note:"呼吸仕事量が軽減",key:true}},
  events:[],
  history:{"S 主訴・症状":"息苦しい。","A アレルギー":"なし。"},
  family:{"家族へ：いつから？":"夜間から急に悪化しました。"},
  exam:{"胸部":"両下肺に湿性ラ音。","四肢":"下腿浮腫あり。"},
  dxOptions:["急性心不全(肺水腫)","気管支喘息発作","肺炎"],
  destOptions:["三次(循環器対応・PCI可能)","二次(一般救急)"],
  aiQ:["心不全を疑った根拠を振り返ってください。"]
}
```
