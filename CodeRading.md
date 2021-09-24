# コードリーディング課題

### 対象
- [settigslogic.rb](https://github.com/settingslogic/settingslogic/blob/master/lib/settingslogic.rb)

### 概要
 settingslogicはRailsアプリケーションに用いられるGemです。設定値をファイルにYAML形式で定義することで、コード内でメソッドとして設定値を呼び出すことができます。

<br>

### settingslogicの使い方
1. 設定値ファイルの作成
2. 設定値ファイルを呼び出すためのクラス定義
3. 設定値を呼び出す

<br>

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
<br>

### 2. 設定値ファイルを呼び出すためのクラス定義
`Settingslogicクラス`を継承して先ほどのファイルパスを記述します。
```ruby
# config/initializers/settings.rb

class Settings < Settingslogic
  source "#{Rails.root}/config/settings.yml"
end
```
<br>

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
<br>
<br>

# ソースコードの説明
### 大まかな流れ
1. Settings.XXXを実行<br>
↓<br>
2. 設定値ファイルを探索し、ハッシュとして読み込む<br>
↓<br>
3. 読み込んだハッシュの`key`を動的にメソッドとして定義<br>
↓<br>
4. `key`に対応した`value`を返す

<br>

### 設定値ファイルの読み込み
```ruby
# settingslogic.rb
  def source(value = nil)
    @source ||= value
  end

# config/initializers/settings.rb
class Settings < Settingslogic
  source "#{Rails.root}/config/settings.yml"
end
```
`source`メソッドによって、`@source`に設定値ファイルのパスが格納されます。

<br>

`Settings.aucfan`を実行した際の流れを順に説明します。
```ruby
irb(main):006:0> Settings.aucfan
=> {"name"=>"株式会社オークファン", "address"=>"東京都品川区大崎...", "service"=>{"guide"=>"タテンポガイド", "robo"=>"オークファンロボ", "pro"=>"aucfan Pro."}}
```

<br>

上記実行時は`aucfan`メソッドはデフォルトで存在しないため、`method_missing`が実行されます。

```ruby
# settingslogic.rb
  def method_missing(name, *args, &block)
    instance.send(name, *args, &block)
  end
```
`method_missing`の引数`name`には`:aucfan`が渡されます。`instance`メソッドが実行されます。

```ruby
# settingslogic.rb
  def instance
    return @instance if @instance
    @instance = new
    create_accessors!
    @instance
  end
```
`instance`メソッドでは、新たにインスタンスを作成し返却します。`new`実行時に`initialize`が呼ばれます。

```ruby
# settingslogic.rb
  def initialize(hash_or_file = self.class.source, section = nil)
    #puts "new! #{hash_or_file}"
    case hash_or_file
    when nil
      raise Errno::ENOENT, "No file specified as Settingslogic source"
    when Hash
      self.replace hash_or_file
    else
      file_contents = open(hash_or_file).read
      hash = file_contents.empty? ? {} : YAML.load(ERB.new(file_contents).result).to_hash #-------(A)
      if self.class.namespace
        hash = hash[self.class.namespace] or return missing_key("Missing setting '#{self.class.namespace}' in #{hash_or_file}")
      end
      self.replace hash #--------(B)
    end
    @section = section || self.class.source  # so end of error says "in application.yml"
    create_accessors!
  end
```
`initialize'メソッドでは、設定値ファイルの内容をハッシュに変換して取り込み、`create_accessors!`に渡します。
(A)で設定値ファイルを参照して、ハッシュに変換しています。
(B)でselfをhashで置き換えています。

```ruby
# settingslogic.rb
  def create_accessors!
    self.each do |key,val|
      create_accessor_for(key)
    end
  end
```
`create_accessors!`メソッドでは、ハッシュの`key`を`create_accessor_for(key)`に渡し、動的にメソッドを作成していきます。

```ruby
# settingslogic.rb
  def create_accessor_for(key, val=nil)
    return unless key.to_s =~ /^\w+$/  # could have "some-setting:" which blows up eval
    instance_variable_set("@#{key}", val)
    self.class.class_eval <<-EndEval
      def #{key}
        return @#{key} if @#{key}
        return missing_key("Missing setting '#{key}' in #{@section}") unless has_key? '#{key}'
        value = fetch('#{key}')
        @#{key} = if value.is_a?(Hash)
          self.class.new(value, "'#{key}' section in #{@section}")
        elsif value.is_a?(Array) && value.all?{|v| v.is_a? Hash}
          value.map{|v| self.class.new(v)}
        else
          value
        end
      end
    EndEval
  end
```
`class_eval'メソッドでクラスメソッドを動的に定義しています。このメソッド内で`key`に対する`value`がハッシュである場合、ハッシュを元に再び`initialize`が実行されます。これを繰り返すことで、YAML形式のネストに沿ってメソッドを定義していきます。

<br>

# 改善点
settignslogicでは、YAMLファイルに既存メソッド名(name, get, methodsなど)で定義してしまうと既存メソッドをオーバーライドしてメソッドを作成してしまうため、下記例のように元のメソッドが使用できなくなる。

```yml
# 不適切なYAMLファイルの設定
defaults: &defaults
  name: "YAMLのnameです。"
#---略---

# 実行結果
irb(main):004:0> Settings.name
=> "YAMLのnameです。"
# 既存メソッドの'.name'では=>"Settings"を返す
```

YAMLファイルの定義が既存メソッドと重複していないかを確認するメソッドを追加することで対応する。
下記のように`yaml_check`メソッドを定義することで、`Settings`クラスで新たに作成されたメソッドに既存メソッド名が含まれていた場合、warningを返すことができる。
```ruby
# settingslogic.rb
  def yaml_check
    if (Settings.methods(false) & Hash.methods).count > 0
      "wartning: #{Settings.methods(false) & Hash.methods}は既存メソッドです。"
    else
      "YAMLファイルは適切です。"
    end
  end

# 実行例
irb(main):001:0> Settings.yaml_check
=> "wartning: [:name]は既存メソッドです。"
```