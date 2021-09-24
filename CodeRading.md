# コードリーディング

### 対象
- [settigslogic.rb](https://github.com/settingslogic/settingslogic/blob/master/lib/settingslogic.rb)

### 概要
 settingslogicはRailsアプリケーションに用いられるGemです。設定値をファイルにYAML形式で定義することで、コード内でメソッドとして設定値を呼び出すことができます。

### settingslogicの使い方
1. 設定値ファイルの作成
2. 設定値ファイルを呼び出すためのクラス定義
3. 設定値を呼び出す


### 1. 設定値ファイルの作成
設定値をYAML形式で記述します。環境ごとに設定できますが、今回はdefaultとして定義しています。
```yml
# config/settings.rb
defaults: &defaults
  aucfan:
    name: "株式会社オークファン"
    address: "東京都品川区大崎..."
    service:
      guide: "タテンポガイド"
      robo: "オークファンロボ"
      pro: "aucfan Pro."

development:
  <<: *defaults

test:
  <<: *defaults

production:
  <<: *defaults

```

### 2. 設定値ファイルを呼び出すためのクラス定義
`Settingslogicクラス`を継承して先ほどのファイルパスを記述します。
```ruby
# config/initializers/settings.rb

class Settings < Settingslogic
  source "#{Rails.root}/config/settings.yml"
end
```

### 3. 設定値を呼び出す
`Settings`の後に設定項目をメソッドチェーンで記述すると、設定値を呼び出すことができます。

```ruby
# console

irb(main):002:0> Settings.aucfan.name
=> "株式会社オークファン"
irb(main):004:0> Settings.aucfan.address
=> "東京都品川区大崎..."
irb(main):005:0> Settings.aucfan.service
=> {"guide"=>"タテンポガイド", "robo"=>"オークファンロボ", "pro"=>"aucfan Pro."}
irb(main):006:0> Settings.aucfan.service.guide
=> "タテンポガイド"
```
