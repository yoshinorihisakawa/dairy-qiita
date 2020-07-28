---
title: 【Bot】毎日Slackに良い記事を届けてくれるDaily Qiita君を作った
tags: Qiita QiitaAPI Slack GoogleAppsScript
author: yoshinori_hisakawa
slide: false
---
# 追記
・2018/1/28
Dairy Qiita君 → Daily Qiita君に変更
一番大事なところミスった。。。。

・2018/1/28
deleteファンクションがうまく機能していなかったので修正


# はじめに

Bot作った背景としては、同期から**「Qiitaの良い記事を自動でSlackに投げてくれたら嬉しいよね！」**と言われ、
確かに面白いと思ったので作ってみました。

何が面白いのかというと、Slack内でのチャンネルに良記事が流れてきたら、
ただのお知らせじゃなくて**「議論のきっかけ」になり、記事を読む以上のことが学べる**といったところです。

# 作ったもの
**GAS + Qiita API + Slack API**で良記事を毎日届けてくれるBot君です。
※良記事=ストック数が3以上 && 直近1週間の中でいいね数が上位30記事　

毎日自動でQiitaの記事をスプレッドシートへ保存してくれます
![スクリーンショット 2019-01-19 17.35.05.png](https://qiita-image-store.s3.amazonaws.com/0/234760/2d1fbee5-d42c-7def-479e-ae8d369651a0.png)

良い記事だけをスクリーニングしてSlackへ送ってくれます
![スクリーンショット 2019-01-19 17.38.55.png](https://qiita-image-store.s3.amazonaws.com/0/234760/7eab6335-e03c-b7d7-ccbd-65588ce7adae.png)

# 要件を決める
まずDaily Qiita君を作るに当たって以下のような要件にしました。

- qiitaに投稿された直近１週間分の中から「いいね」が多い記事をSlackに投稿したい
- Slackのチャネル（goチャネル、javaチャネルなど）毎にタグづけされた対象の記事を振り分けたい
- 良記事30件くらいのデータのみをSlackに投稿したい（随時調整）
- 不要なデータはスプレッドシートから削除したい
- 同じ記事は送られたくない。一度送ったデータは再度Slackに送りたくない
- 記事のいいね数/ストック数、URL情報をSlackに送りたい
- 毎朝９時など通勤時間帯にSlackに送りたい
- 構想から実現までのリミットは1日（個人的に時間はかけたくない）

# コード
コピペしたい人用にgithubにコードを上げています。
https://github.com/yoshinorihisakawa/dairy-qiita

実装する前に処理の流れざっと考えてみました。

- qiitaの記事を直近１週間分取得する
    - stock数 > 3　以上などにして記事数を絞る
    - 毎日起動するので、重複したデータはスプレッドシートに書き込まない
    - 最後に追記する形で記事をスプレッドシートに書き込んでいく
- ストック数を取得する
    - ストック数を取得するには、記事のIDで再度APIを叩くしか方法はなかった
- 取得した記事をソートする
    - 良い記事を上位30件までにしたかったのでいい記事順にソートする
- Slackに記事を送信する
    - ソート済みなので上から順に30件Slack送る
    - タグ別にチャンネルを振り分ける
    - 送ったデータには送り済みフラグをつける
- 直近１週間でない記事は削除する
    - データが多くなると処理も重くなるので

あとは実装あるのみ。1日で終わらせたかったので、動くことを優先させたため、
綺麗なコードとは言えません。

```javascript

function main() {
  inputItems()
  getItemStock()
  sort()
  postSlackMessage()
  deleteOneWeek()
}

function inputItems() {
  const COL_TITLE_ID = 1;
  const COL_TITLE = 2;
  const COL_URL = 3;
  const COL_TAGS = 4;
  const COL_LIKE = 9;
  const COL_STOCK = 10;
  const COL_ISSEND = 12;
  const COL_GETDATE = 13;
  
  // stockが3以上 && 直近１週間前のデータ
  const API_ENDPOINT = 'https://qiita.com/api/v2/items?page=1&per_page=100&query=stocks%3A%3E6+created%3A%3E';
  // １週間前を指定する
  const CREATED_AT = Moment.moment().add(-7,'d').format('YYYY-MM-DD');
  try {
    var res = UrlFetchApp.fetch(API_ENDPOINT + CREATED_AT);
    var json = JSON.parse(res.getContentText());
    
    var sheet = SpreadsheetApp.getActiveSheet();
    
    // 書き込み関連の処理
    json.forEach(function(item){
      lastRow = sheet.getLastRow();

      // 重複チェック　重複していた場合は処理をスキップさせる
      var ids = sheet.getRange(2, COL_TITLE_ID, lastRow -1).getValues();
      var isDuplicated = ids.some(function(array, i) {
        return (array[0] === item["id"]);
      });
      if (isDuplicated === true) {
        Logger.log(item["id"])
        return;
      }
      
      // 最後の行に追記する形で行書き込み
      rowToWrite = lastRow + 1   
      sheet.getRange(rowToWrite, COL_TITLE_ID).setValue(item["id"]);
      sheet.getRange(rowToWrite, COL_TITLE).setValue(item["title"]);
      sheet.getRange(rowToWrite, COL_URL).setValue(item["url"]);
      var tags = item["tags"]
      tags.forEach(function(tag, i){
        sheet.getRange(rowToWrite, COL_TAGS + i).setValue(tags[i]["name"]);
      });
      sheet.getRange(rowToWrite, COL_LIKE).setValue(item["likes_count"]);
      var now = Moment.moment().format('YYYYMMDD');  
      sheet.getRange(rowToWrite, COL_GETDATE).setValue(now);
    });
  } catch (ex) {
      Logger.log(ex)
  }
}

function getItemStock() {
  const COL_TITLE_ID = 1;
  const COL_TITLE = 2;
  const COL_URL = 3;
  const COL_TAGS = 4;
  const COL_LIKE = 9;
  const COL_STOCK = 10;
  const COL_ISSEND = 12;
  const COL_GETDATE = 13;
  
  var sheet = SpreadsheetApp.getActiveSheet();
  lastRow = sheet.getLastRow();
  
  var ids = sheet.getRange(2, COL_TITLE_ID, lastRow -1).getValues();
  const API_ENDPOINT_REPLACE = 'https://qiita.com/api/v2/items/{id}/stockers?page=1&per_page=100';
  // 100以上は取得しない。100以上のは送る
  for(var i = 0;　i < ids.length; i ++) {
    var API_ENDPOINT = API_ENDPOINT_REPLACE.replace('{id}',ids[i]);
    
    try {
      var res = UrlFetchApp.fetch(API_ENDPOINT);
      var js = JSON.parse(res.getContentText());
      var stockNum = Object.keys(js).length;
      sheet.getRange(i + 2, COL_STOCK).setValue(stockNum);
    } catch (ex) {
      Logger.log(ex)
    }
    
  }
}

function sort() {
  const COL_LIKE = 9;
  const COL_STOCK = 10;
  var sheet = SpreadsheetApp.getActiveSheet();
  var lastRow = sheet.getLastRow();
  var lastCol = sheet.getLastColumn();
  sheet.getRange(2, 1, lastRow, lastCol).sort({column: COL_STOCK, ascending: false});
  sheet.getRange(2, 1, lastRow, lastCol).sort({column: COL_LIKE, ascending: false});
}

function postSlackMessage() { const COL_TITLE_ID = 1;
  const COL_TITLE = 2;
  const COL_URL = 3;
  const COL_TAGS = 4;
  const COL_LIKE = 9;
  const COL_STOCK = 10;
  const COL_ISSEND = 12;
  const COL_GETDATE = 13;
  
  var sheet = SpreadsheetApp.getActiveSheet();
  
                             
  var token = PropertiesService.getScriptProperties().getProperty('SLACK_ACCESS_TOKEN');
 
  var slackApp = SlackApp.create(token);
                             
  var keyValue = { 'Go': '#com_golang', 
                   'アジャイル': '#z_agile',
                   'CI': '#z_cicd',
                   'JavaScript': '#z_javascript',
                   'Ruby': '#z_ruby',
                   'AWS': '#z_aws',
                   'docker': '#z_docker',
                   'Java': '#z_java',
                 };
                             
  for(var i = 2;　i < 31; i ++) {
    // その行がすでに送りずみならcontinue
    var isSend = sheet.getRange(i, COL_ISSEND).getValue()
    if (isSend === true) {
      continue;
    }
    // タグ１から5まで
     for(var t = 0;　t < 5; t ++) {
       var tag = sheet.getRange(i, t + COL_TAGS).getValue()
       Object.keys(keyValue).forEach(function(kv){
         //valueを配列にする
         var ary = kv.split(',');
         
         var exist = ary.some(function(str, i, data) {
           return (str === tag);
         });
         
         if (exist) {
           var title = sheet.getRange(i, COL_TITLE).getValue()
           var url = sheet.getRange(i, COL_URL).getValue()
           var like = sheet.getRange(i, COL_LIKE).getValue()
           var stock = sheet.getRange(i, COL_STOCK).getValue()
                  
           var options = {
             channelId: keyValue[kv], //チャンネル名
             userName: "daily qiita君", //投稿するbotの名前
             message: title + "\n" + url + "\n" + "いいね数 = " + like  + " / ストック数 = " + stock  //投稿するメッセージ
           };
           
           slackApp.postMessage(options.channelId, options.message, {username: options.userName});
   
           // 最後に送りずみマークのtrueをつける
           sheet.getRange(i, COL_ISSEND).setValue(true);
           
           return;
         }
       });
     }
  }                   
}

function deleteOneWeek() {
  const COL_GETDATE = 13;
  // １週間前のデータは削除
  var DELETED_AT = Moment.moment().add(-8,'d').format('YYYYMMDD');
  var sheet = SpreadsheetApp.getActiveSheet();
    
  // 値の比較
  var lastRow = sheet.getLastRow();
  var dates = sheet.getRange(2, COL_GETDATE, lastRow -1).getValues();
  var row = 0
  dates.forEach(function(date, i){
    if (DELETED_AT == date.toString()) {
      sheet.deleteRow(row + 2);
      row = row - 1
    }
    row = row + 1
  });
}
```

# daily-qiita君をあなたの環境で動かすには？
**1. Googleのアドオン追加でGASを使えるようにします。**
以下のURLを参照
https://developers.google.com/apps-script/

**2. スクリプト × スプレッドシートの準備**
スプレッドシート準備：　Googleドライブ > 新規　> Googleスプレッドシート
スクリプト準備: Googleスプレッドシートのツール > スクリプトエディタ

**3. ライブラリの追加**
Moment: 時刻を扱うライブラリ
https://tonari-it.com/gas-moment-js-moment/

SlackApp: slackを扱うライブラリ
https://qiita.com/soundTricker/items/43267609a870fc9c7453

**4. スプレッドシートに雛形を用意**
手書きでもいいですが、こちらのスクリプトを貼って、動かせばスプレッドシートに雛形が完成します。

```javascript

function　init() {
  // シートの名前作成
  const COL_TITLE_ID = 1;
  const COL_TITLE = 2;
  const COL_URL = 3;
  const COL_TAGS1 = 4;
  const COL_TAGS2 = 5;
  const COL_TAGS3 = 6;
  const COL_TAGS4 = 7;
  const COL_TAGS5 = 8;
  const COL_LIKE = 9;
  const COL_STOCK = 10;
  const COL_ISSEND = 12;
  const COL_GETDATE = 13;

  var sheet = SpreadsheetApp.getActiveSheet();
  //シート初期化
  //sheet.clear();
  sheet.getRange(1, COL_TITLE_ID).setValue("記事ID");
  sheet.getRange(1, COL_TITLE).setValue("タイトル");
  sheet.getRange(1, COL_URL).setValue("URL");
  sheet.getRange(1, COL_TAGS1).setValue("タグ１");
  sheet.getRange(1, COL_TAGS2).setValue("タグ２");
  sheet.getRange(1, COL_TAGS3).setValue("タグ３");
  sheet.getRange(1, COL_TAGS4).setValue("タグ４");
  sheet.getRange(1, COL_TAGS5).setValue("タグ５");
  sheet.getRange(1, COL_LIKE).setValue("いいね数");
  sheet.getRange(1, COL_STOCK).setValue("ストック数");
  sheet.getRange(1, COL_ISSEND).setValue("status");
  sheet.getRange(1, COL_GETDATE).setValue("取得日");
}
```

**5. コードをコピペ**
そのまま貼り付けたらおっけー！！
https://github.com/yoshinorihisakawa/dairy-qiita

**6. スラックBotのAPIの登録**
https://qiita.com/ykhirao/items/3b19ee6a1458cfb4ba21

**7. 定期実行設定**
https://tonari-it.com/gas-trigger-set/

# 注意
・初回の承認手続き
承認の要求が出てきたときは以下の手順で
https://www.virment.com/step-allow-google-apps-script/

# 最後に
GASを使えば結構簡単に実現したいことを実現できると今更ながら思いました。
他にもおもしそうなことがあれば、思うだけではなく実現していきたいです！
