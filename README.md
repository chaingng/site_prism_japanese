# SitePrismの日本語訳
https://github.com/natritmeyer/site_prism

# SitePrism
_A Page Object Model DSL for Capybara_

SitePrismは、あなたのWebサイトをPage Object Modelパターンで記述し、自動テストにCapybaraを使った、シンプル、クリーンでセマンティックなDSLを提供します。 

整形されたドキュメントはこちら: http://rdoc.info/gems/site_prism/frames

[![Build Status](https://travis-ci.org/natritmeyer/site_prism.png)](https://travis-ci.org/natritmeyer/site_prism)

あなたのproject/companyをこちらに追加してください: https://github.com/natritmeyer/site_prism/wiki/Who-is-using-SitePrism

## 概要

SitePrismの使用方法の概要:

```ruby
# define our site's pages

class Home < SitePrism::Page
  set_url "/index.htm"
  set_url_matcher /google.com\/?/

  element :search_field, "input[name='q']"
  element :search_button, "button[name='btnK']"
  elements :footer_links, "#footer a"
  section :menu, MenuSection, "#gbx3"
end

class SearchResults < SitePrism::Page
  set_url_matcher /google.com\/results\?.*/

  section :menu, MenuSection, "#gbx3"
  sections :search_results, SearchResultSection, "#results li"

  def search_result_links
    search_results.map {|sr| sr.title['href']}
  end
end

# define sections used on multiple pages or multiple times on one page

class MenuSection < SitePrism::Section
  element :search, "a.search"
  element :images, "a.image-search"
  element :maps, "a.map-search"
end

class SearchResultSection < SitePrism::Section
  element :title, "a.title"
  element :blurb, "span.result-description"
end

# now for some tests

When /^I navigate to the google home page$/ do
  @home = Home.new
  @home.load
end

Then /^the home page should contain the menu and the search form$/ do
  @home.wait_for_menu # menu loads after a second or 2, give it time to arrive
  expect(@home).to have_menu
  expect(@home).to have_search_field
  expect(@home).to have_search_button
end

When /^I search for Sausages$/ do
  @home.search_field.set "Sausages"
  @home.search_button.click
end

Then /^the search results page is displayed$/ do
  @results_page = SearchResults.new
  expect(@results_page).to be_displayed
end

Then /^the search results page contains 10 individual search results$/ do
  @results_page.wait_for_search_results
  expect(@results_page).to have_search_results count: 10
end

Then /^the search results contain a link to the wikipedia sausages page$/ do
  expect(@results_page.search_result_links).to include "http://en.wikipedia.org/wiki/Sausage"
end
```

次に詳細を...

## セットアップ
### インストール
SitePrismのインストール:

```bash
gem install site_prism
```

### CucumberでのSitePrismの利用
cucumberを使っている場合は, 以下が必要:

```ruby
require 'capybara'
require 'capybara/cucumber'
require 'selenium-webdriver'
require 'site_prism'
```

### RSpecでのSitePrismの利用
rspecを使う場合は以下が必要:

```ruby
require 'capybara'
require 'capybara/rspec'
require 'selenium-webdriver'
require 'site_prism'
```

## Page Object Modelのイントロダクション

Page Object Modelはテストに使うWebサイトのユーザインタフェースを抽象化することを目的としたテスト自動化パターンである.
最も一般的な方法はページをクラスとしてモデル化し、テストでそのインスタンスを使う方法である.

クラスがページを表してページの要素がメソッドにより表されるとすると、
メソッドが呼ばれることでその要素に対するアクション(clicked, set text value)、もしくは問い合わせされた状態(is it enabled? visible?)が返される.

SitePrismはこのコンセプトに基づいていて、さらに単一ページで複数回現れる、
または複数ページで繰り返し現れるセクションのモデル化も可能になる.

## Pages

名前から想像される通り、PagesはPage Object Modelの中心的な存在になる. 
SitePrismではPagesをどのようにモデル化するかを説明する:

### Page Modelの作成
何も定義されていない最もシンプルなページ。
以下の例からHomeのpage作成を開始する:

```ruby
class Home < SitePrism::Page
end
```

上の例は名前だけの定義で使える例ではない.

### URLの追加
ページはたいていURLをもつ。ページナビゲーションを行うにはURLを設定する必要がある:

```ruby
class Home < SitePrism::Page
  set_url "http://www.google.com"
end
```

Capybaraの `app_host` をセットすることで次のようにも定義できる:

```ruby
class Home < SitePrism::Page
  set_url "/home.htm"
end
```

URLの設定はオプションである - ページへの直接の遷移が必要なときのみ必要になる.
ホームページやログインページにURLをセットするが、検索結果にはおそらくいらないということが理解できる.

#### URLパラメータの使用
SitePrism はAddressable gemを使っているのでパラメータURLも扱うことができる:

```ruby
class UserProfile < SitePrism::Page
  set_url "/users{/username}"
end
```

...そしてより複雑な例:

```ruby
class Search < SitePrism::Page
  set_url "/search{?query*}"
end
```

https://github.com/sporkmonger/addressable にパラメータURLの詳細が記載されている.

### Pageナビゲーション
URLをセットすると ( `set_url`), `#load` によって直接ページ遷移が可能:

```ruby
@home_page = Home.new
@home_page.load
```

#### URLパラメータによるPageナビゲーション
`#load` メソッドはパラメータをもちURLに適用できる:

```ruby
class UserProfile < SitePrism::Page
  set_url "/users{/username}"
end

@user_profile = UserProfile.new
@user_profile.load #=> /users
@user_profile.load(username: 'bob') #=> loads /users/bob
```

...そして...

```ruby
class Search < SitePrism::Page
  set_url "/search{?query*}"
end

@search = Search.new
@search.load(query: 'simple') #=> loads /search?query=simple
@search.load(query: {'color'=> 'red', 'text'=> 'blue'}) #=> loads /search?color=red&text=blue
```

Capybara driverがPageクラスに設定されたURLに対して遷移させていることがわかる

https://github.com/sporkmonger/addressable にパラメータURLの詳細が記載されている.

### あるページが表示されることの確認
自動テストはしばしば特定のページが表示されていることを確認する.
SitePrismは自動でURLをパースして、テンプレートが指定する要素が現在表示しているページと一致するか確認する:

```ruby
class Account < SitePrism::Page
  set_url "/accounts/{id}{?query*}"
end
```

次のコードはテストをパスする:

```ruby
@account_page = Account.new
@account_page.load(id: 22, query: { token: "ca2786616a4285bc" })

expect(@account_page.current_url).to end_with "/accounts/22?token=ca2786616a4285bc"
expect(@account_page).to be_displayed
```

`#displayed?` を呼ぶことで現在のブラウザのURLがページテンプレートと一致するかどうか確認できる.

#### テンプレートURLにおけるパラメータ値の特定

時々、現在のURLがテンプレートにマッチしていている以上のことを確かめたいときがある.
前の例にて、account number 22がブラウザでロードされることを保証するためには以下をassertする:

```ruby
expect(@account_page).to be_displayed(id: 22)
```

正規表現を使うことも可能である.
この例ではaccount idが2で終わることを確かめている:

```ruby
expect(@account_page).to be_displayed(id: /2\z/)
```

#### テストにおけるテンプレートURL中のマッチング

`displayed?` オプションが十分に確かめたいことを満たしていないとき、
`url_matches`によってページのURLテンプレートと現在のURLを比較することができる:

```ruby
@account_page = Account.new
@account_page.load(id: 22, query: { token: "ca2786616a4285bc", color: 'irrelevant' })

expect(@account_page).to be_displayed(id: 22)
expect(@account_page.url_matches['query']['token']).to eq "ca2786616a4285bc"
```

#### basic regexp matchersの使用
SitePrismのビルドインURLマッチングでは十分でない場合、`set_url_matcher`を呼ぶことで
以前サポートしていた正規表現ベースのURLマッチャーをオーバーライドして使用することが可能:

```ruby
class Account < SitePrism::Page
  set_url_matcher %r{/accounts/\d+}
end
```

#### Page表示のテスト
`#displayed?` メソッドはテストで利用可能:

```ruby
Then /^the account page is displayed$/ do
  expect(@account_page).to be_displayed
  expect(@some_other_page).not_to be_displayed
end
```

### 現在のURLの取得
SitePrismは現在のページURLを取得可能。
次のようにして取得:

```ruby
class Account < SitePrism::Page
end

@account = Account.new
#...
@account.current_url #=> "http://www.example.com/account/123"
expect(@account.current_url).to include "example.com/account/"
```

### タイトルの取得
ページタイトルの取得は難しくない:

```ruby
class Account < SitePrism::Page
end

@account = Account.new
#...
@account.title #=> "Welcome to Your Account"
```

### HTTP vs. HTTPS
一般に現在のURLが'https'で始まるかどうかチェックすることで簡単にセキュアなページかそうでないかが判断できる。
SitePrism `secure?` メソッドにより現在のURLが'https'で始まればtrue、そうでなければfalseを返す:

```ruby
class Account < SitePrism::Page
end

@account = Account.new
#...
@account.secure? #=> true/false
expect(@account).to be_secure
```

## 要素
Pageは単一の要素や要素の集合、様々な要素(text fields, buttons, combo boxes, etc)から成る.
単一の要素は例えばsearch field or a company logo imageがある.
要素集合は例えばitems in any sort of list, eg: menu items,
images in a carouselなどがある.

### 単一の要素
単一の要素を紐づけるためには、ページの一部として定義する必要がある。
SitePrismでは簡単に定義可能:

```ruby
class Home < SitePrism::Page
  element :search_field, "input[name='q']"
end
```

Here we're adding a search field to the Home page. 
`element`メソッドは２つの引数をもつ。要素名としてのシンボルとcssセレクタ文字列.

#### 単一要素へのアクセス
`element` メソッドはPageクラスのインスタンスに多くのメソッドを追加する.
最初のメソッドはelement名のメソッドになる:

```ruby
class Home < SitePrism::Page
  set_url "http://www.google.com"

  element :search_field, "input[name='q']"
end
```

... 次のようにsearch fieldを取り扱うことができる:

```ruby
@home = Home.new
@home.load

@home.search_field #=> will return the capybara element found using the selector
@home.search_field.set "the search string" #=> since search_field returns a capybara element, you can use the capybara API to deal with it
@home.search_field.text #=> standard method on a capybara element; returns a string
```

#### 要素が存在することのテスト

`element` メソッドの追加により、Pageクラスに`has_<element name>?` メソッドも追加される:

```ruby
class Home < SitePrism::Page
  set_url "http://www.google.com"

  element :search_field, "input[name='q']"
end
```

... 要素の存在を次のようにテスト可能:

```ruby
@home = Home.new
@home.load
@home.has_search_field? #=> returns true if it exists, false if it doesn't
```

...こちらはよいテストコード:

```ruby
Then /^the search field exists$/ do
  expect(@home).to have_search_field
end
```

#### 要素が存在しないことのテスト

ページに要素が存在しないことをテストするために、単に`#not_to have_search_field`を呼ぶことはできない. 
SitePrism は`#has_no_<element>?` メソッドにより要素が存在しないことをテストする:

```ruby
@home = Home.new
@home.load
@home.has_no_search_field? #=> returns true if it doesn't exist, false if it does
```

...こちらはよいテストコード:

```ruby
Then /^the search field exists$/ do
  expect(@home).to have_no_search_field #NB: NOT => expect(@home).not_to have_search_field
end
```

#### 要素が存在するまで待つ

 `element` の追加によって、 `wait_for_<element_name>` メソッドも追加される.
このメソッドにより、Capybaraのデフォルトwait time だけ要素が現れるのを待つ. 
wait timeはカスタム可能:

```ruby
class Home < SitePrism::Page
  set_url "http://www.google.com"

  element :search_field, "input[name='q']"
end
```

... search fieldが現れるのを次のようにテストできる:

```ruby
@home = Home.new
@home.load
@home.wait_for_search_field
# or...
@home.wait_for_search_field(10) #will wait for 10 seconds for the search field to appear
```

#### 要素が見えるまで待つ

`element`を呼ぶことにより、`wait_until_<element_name>_visible`メソッドも追加される.
このメソッドを呼ぶことでCapybaraのデフォルトwait timeだけ要素がvisibleになる(existanceではない)のを待つことができる
waitの秒数を定義することでカスタマイズすることも可能:

```ruby
@home.wait_until_search_field_visible
# or...
@home.wait_until_search_field_visible(10)
```

#### CSS Selectors vs. XPath Expressions
上の例はCSSセレクタの例だが、XPathもどこでも使用可能:

```ruby
class Home < SitePrism::Page
  # CSS Selector:
  element :first_name, "div#signup input[name='first-name']"

  #same thing as an XPath expression:
  element :first_name, :xpath, "//div[@id='signup']//input[@name='first-name']"
end
```

#### 要素に対するメソッドの概要:

こちらが与えられたとき:

```ruby
class Home < SitePrism::Page
  element :search_field, "input[name='q']"
end
```

...次のメソッドが利用可能になる:

```ruby
@home.search_field
@home.has_search_field?
@home.has_no_search_field?
@home.wait_for_search_field
@home.wait_for_search_field(10)
@home.wait_until_search_field_visible
@home.wait_until_search_field_visible(10)
@home.wait_until_search_field_invisible
@home.wait_until_search_field_invisible(10)

```

### 要素の集合
例えば名前のリストのように、単一の要素ではなく要素集合を用いたいときがある。
SitePrismではPageクラスにおいて`elements`メソッドを利用する:

```ruby
class Friends < SitePrism::Page
  elements :names, "ul#names li a"
end
```

`element` メソッドと同様に、`elements` メソッドは２つの引数を持つ:
１つ目は要素名としてのシンボル、２つ目はCapybara要素配列を返すCSSセレクタである.

#### 要素へのアクセス

`element`と同様に, `elements`メソッドはPageクラスにいくつかメソッドを追加する.
1つはCSSセレクタにマッチするCapybara要素配列を返す配列である:

```ruby
class Friends < SitePrism::Page
  elements :names, "ul#names li a"
end
```

次のようにアクセス可能:

```ruby
@friends_page = Friends.new
# ...
@friends_page.names #=> [<Capybara::Element>, <Capybara::Element>, <Capybara::Element>]
```

また配列に対してできる通常の操作も利用可能:


```ruby
@friends_page.names.each {|name| puts name.text}
expect(@friends_page.names.map {|name| name.text}.to eq ["Alice", "Bob", "Fred"]
expect(@friends_page.names.size).to eq 3
expect(@friends_page).to have(3).names
```

#### 要素集合のテスト
#### 
`element`メソッドと同様に,`elements`メソッドも `has_<element collection name>?`で
呼ぶことのできる存在確認を行うメソッドを追加する.少なくとも配列中の１つでも要素が存在したらtrueを返し、そうでなければfalseを返す:

```ruby
class Friends < SitePrism::Page
  elements :names, "ul#names li a"
end
```

... 次のメソッドも利用可能:

```ruby
@friends_page.has_names? #=> returns true if at least one element is found using the relevant selector
```

...テストすることも可能:

```ruby
Then /^there should be some names listed on the page$/ do
  expect(@friends_page).to have_names
end
```

#### 要素集合を待つ
単一要素と同様に、要素集合も出現するまで待つためのメソッドがある. 
`elements`メソッドは`wait_for_<element collection name>` も追加し、
このメソッドを呼ぶことで,少なくとも１要素がセレクタにマッチするまでCapybraのデフォルトwait timeだけ待つ:

```ruby
class Friends < SitePrism::Page
  elements :names, "ul#names li a"
end

```

... このようにnamesが存在するまで待つことができる:

```ruby
@friends_page.wait_for_names
```

また、待ち時間もカスタマイズすることができる:

```ruby
@friends_page.wait_for_names(10)
```

#### 要素がvisible or invisibleになるのを待つ
Like the individual elements, calling the `elements` method will create
two methods: `wait_until_<elements_name>_visible` and
`wait_until_<elements_name>_invisible`. Calling these methods will cause
your test to wait for the elements to become visible or invisible. Using
the above example:

```ruby
@friends_page.wait_until_names_visible
# and...
@friends_page.wait_until_names_invisible
```

It is possible to wait for a specific amount of time instead of using
the default Capybara wait time:

```ruby
@friends_page.wait_until_names_visible(5)
# and...
@friends_page.wait_until_names_invisible(7)
```

### すべてのマッピングした要素がページ中に存在するかチェックする

Throughout my time in test automation I keep getting asked to provide the
ability to check that all elements that should be on the page are on the
page. Why people would want to test this, I don't know. But if that's
what you want to do, SitePrism provides the `#all_there?` method that
will return true if all mapped elements (and sections... see below) are
present in the browser, false if they're not all there.

```ruby
@friends_page.all_there? #=> true/false

# and...

Then /^the friends page contains all the expected elements$/ do
  expect(@friends_page).to be_all_there
end

```

## セクション
SitePrism allows you to model sections of a page that appear on multiple
pages or that appear a number of times on a page separately from Pages.
SitePrismはこれを実現するためにSectionを提供する.

### 単一のセクション
In the same way that SitePrism provides `element` and `elements`, it
provides `section` and `sections`. The first returns an instance of a
page section, the second returns an array of section instances, one for
each capybara element found by the supplied css selector. What follows
is an explanation of `section`.


#### セクションの定義
sectionはページと似ており、SitePrismクラスを継承する:

```ruby
class MenuSection < SitePrism::Section
end
```

ここまででは,セクションは何もしない.

#### Pageへのセクションの追加
SitePrismが動いているPagesにセクションをインクルードする. 
上の`MenuSection`をPageにインクルードした例:

```ruby
class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end
```

The way to add a section to a page (or another section -
SitePrism allows adding sections to sections) is to call the `section`
method. It takes 3 arguments: the first is the name of the section as
referred to on the page (sections that appear on multiple pages can be
named differently). The second argument is the class of which an
instance will be created to represent the page section, and the third
argument is a css selector that identifies the root node of the section
on this page (note that the css selector can be different for different
pages as the whole point of sections is that they can appear in
different places on different pages).

#### ページ中のセクションへのアクセス

The `section` method (like the `element` method) adds a few methods to
the page or section class it was called against. The first method that
is added is one that returns an instance of the section, the method name
being the first argument to the `section` method. Here's an example:

```ruby
# the section:

class MenuSection < SitePrism::Section
end

# the page that includes the section:

class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end

# the page and section in action:

@home = Home.new
@home.menu #=> <MenuSection...>
```

When the `menu` method is called against `@home`, an instance of
`MenuSection` (the second argument to the `section` method) is returned.
The third argument that is passed to the `section` method is the css
selector that will be used to find the root element of the section; this
root node becomes the 'scope' of the section.

The following shows that though the same section can appear on multiple
pages, it can take a different root node:

```ruby
# define the section that appears on both pages

class MenuSection < SitePrism::Section
end

# define 2 pages, each containing the same section

class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end

class SearchResults < SitePrism::Page
  section :menu, MenuSection, "#gbx48"
end
```

You can see that the `MenuSection` is used in both the `Home` and
`SearchResults` pages, but each has slightly different root node. The
capybara element that is found by the css selector becomes the root node
for the relevant page's instance of the `MenuSection` section.

#### セクションへの要素追加

単純に、ページへ要素を追加するのと同じ方法で追加できる:

```ruby
class MenuSection < SitePrism::Section
  element :search, "a.search"
  element :images, "a.image-search"
  element :maps, "a.map-search"
end
```

Note that the css selectors used to find elements are searched for
within the scope of the root element of that section. The search for the
element won't be page-wide but it will only look in the section.

ページにセクションが追加されると...

```ruby
class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end
```

...セクションの要素は次のようにアクセスできるようになる:

```ruby
@home = Home.new
@home.load

@home.menu.search #=> returns a capybara element representing the link to the search page
@home.menu.search.click #=> clicks the search link in the home page menu
@home.menu.search['href'] #=> returns the value for the href attribute of the capybara element representing the search link
@home.menu.has_images? #=> returns true or false based on whether the link is present in the section on the page
@home.menu.wait_for_images #=> waits for capybara's default wait time until the element appears in the page section

```

...いくつかのよいテストコード:

```ruby
Then /^the home page menu contains a link to the various search functions$/ do
  expect(@home.menu).to have_search
  expect(@home.menu.search['href']).to include "google.com"
  expect(@home.menu).to have_images
  expect(@home.menu).to have_maps
end
```

##### ブロックを利用したセクション要素へのアクセス

Sections have a `within` method that allows scoped access to the section's elements inside a block.  This is similar to Capybara's `within` method and allows for shorter test code particularly with nested sections.
Some of this test code can be made a little prettier by simply passing a block in.

```ruby
Then /^the home page menu contains a link to the various search functions$/ do
  @home.menu do |menu|
    expect(menu).to have_search
    expect(menu.search['href']).to include "google.com"
    expect(menu).to have_images
    expect(menu).to have_maps
  end
end
```

#### セクションの親を取得

Sectionに対して、その親(ページ、もしくはサブセクションとなっているセクション)を取得することができる.:

```ruby
class MySubSection < SitePrism::Section
  element :some_element, "abc"
end

class MySection < SitePrism::Section
  section :my_subsection, MySubSection, "def"
end

class MyPage < SitePrism::Page
  section :my_section, MySection, "ghi"
end
```

...`#parent` を呼ぶことで次が返ってくる:

```ruby
@my_page = MyPage.new
@my_page.load

@my_page.my_section.parent #=> returns @my_page
@my_page.my_section.my_subsection.parent #=> returns @my_section
```

#### セクションの親ページを取得
Sectionが属しているPageを取得することが可能:

```ruby
class MenuSection < SitePrism::Section
  element :search, "a.search"
  element :images, "a.image-search"
  element :maps, "a.map-search"
end

class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end
```

...Sectionの親ページを取得できる:

```ruby
@home = Home.new
@home.load
@home.menu.parent_page #=> returns @home
```

#### セクションの存在をテスト

Just like elements, it is possible to test for the existence of a
section. The `section` method adds a method called `has_<section name>?`
to the page or section it's been added to - same idea as what the
`has_<element name>?` method. Given the following setup:

```ruby
class MenuSection < SitePrism::Section
  element :search, "a.search"
  element :images, "a.image-search"
  element :maps, "a.map-search"
end

class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end
```

... you can check whether the section is present on the page or not:

```ruby
@home = Home.new
#...
@home.has_menu? #=> returns true or false
```

同様に、こちらはよいテストコード:

```ruby
expect(@home).to have_menu
expect(@home).not_to have_menu
```

#### セクションが出現するのを待つ

Another method added to the page or section by the `section` method is
`wait_for_<section name>`. Similar to what `element` does, this method
waits for the section to appear - the test will wait up to capybara's
default wait time until the root node of the element exists on the
page/section that our section was added to. Given the following setup:

```ruby
class MenuSection < SitePrism::Section
  element :search, "a.search"
  element :images, "a.image-search"
  element :maps, "a.map-search"
end

class Home < SitePrism::Page
  section :menu, MenuSection, "#gbx3"
end
```

... このようにページ中にmenu sectionが現れるのを待つことができる:

```ruby
@home.wait_for_menu
@home.wait_for_menu(10) # waits for 10 seconds instead of capybara's default timeout
```

#### セクションがvisible or invisibleになるのを待つ
要素と同様に、Sectionがvisibleまたはinvisibleになるのを待つことができる. 
`section`メソッドは関連するpageもしくはsectionに２つのメソッドを追加する:
`wait_until_<section_name>_visible` と`wait_until_<section_name>_invisible`. 
上の例では、次のように使うことができる:

```ruby
@home = Home.new
@home.wait_until_menu_visible
# and...
@home.wait_until_menu_invisible
```

Again, as for an element, it is possible to give a specific amount of
time to wait for visibility/invisibility of a section. Here's how:

```ruby
@home = Home.new
@home.wait_until_menu_visible(5)
# and...
@home.wait_until_menu_invisible(3)
```

#### セクション内のセクション
セクションはページだけでなく、セクションへのセクションへのセクションへのセクションへのネストも可能!:

```ruby

# define a page that contains an area that contains a section for both logging in and registration, then modelling each of the sub sections separately

class Login < SitePrism::Section
  element :username, "#username"
  element :password, "#password"
  element :sign_in, "button"
end

class Registration < SitePrism::Section
  element :first_name, "#first_name"
  element :last_name, "#last_name"
  element :next_step, "button.next-reg-step"
end

class LoginRegistrationForm < SitePrism::Section
  section :login, Login, "div.login-area"
  section :registration, Registration, "div.reg-area"
end

class Home < SitePrism::Page
  section :login_and_registration, LoginRegistrationForm, "div.login-registration"
end

# how to login (fatuous, but demonstrates the point):

Then /^I sign in$/ do
  @home = Home.new
  @home.load
  @home.wait_for_login_and_registration
  expect(@home).to have_login_and_registration
  expect(@home.login_and_registration).to have_username
  @home.login_and_registration.login.username.set "bob"
  @home.login_and_registration.login.password.set "p4ssw0rd"
  @home.login_and_registration.login.sign_in.click
end

# how to sign up:

When /^I enter my name into the home page's registration form$/ do
  @home = Home.new
  @home.load
  expect(@home.login_and_registration).to have_first_name
  expect(@home.login_and_registration).to have_last_name
  @home.login_and_registration.first_name.set "Bob"
  # ...
end
```

#### 匿名セクション集合

セクションを要素の名前空間以上のものとして使いたい時、
そして再利用しないときはブロックを使って匿名セクションを定義するのが便利:

```ruby
class Home < SitePrism::Page
  section :menu, '.menu' do
    element :title, '.title'
    elements :items, 'a'
  end
end
```
このコードは匿名セクションを作成し、普通のセクションのように使うことができる:

```ruby
@home = Home.new
expect(@home.menu).to have_title
```

### セクション集合

An individual section represents a discrete section of a page, but often
sections are repeated on a page, an example is a search result listing -
each listing contains a title, a url and a description of the content.
It makes sense to model this only once and then to be able to access
each instance of a search result on a page as an array of SitePrism
sections. To achieve this, SitePrism provides the `sections` method that
can be called in a page or a section.

The only difference between `section` and `sections` is that whereas the
first returns an instance of the supplied section class, the second
returns an array containing as many instances of the section class as
there are capybara elements found by the supplied css selector. This is
better explained in code :)

#### セクション集合をページ(or他のセクション)に追加

Given the following setup:

```ruby
class SearchResultSection < SitePrism::Section
  element :title, "a.title"
  element :blurb, "span.result-decription"
end

class SearchResults < SitePrism::Page
  sections :search_results, SearchResultSection, "#results li"
end
```

... it is possible to access each of the search results:

```ruby
@results_page = SearchResults.new
# ...
@results_page.search_results.each do |search_result|
  puts search_result.title.text
end
```

... which allows for pretty tests:

```ruby
Then /^there are lots of search_results$/ do
  expect(@results_page.search_results.size).to eq 10
  @results_page.search_results.each do |search_result|
    expect(search_result).to have_title
    expect(search_result.blurb.text).not_to be_nil
  end
end
```

The css selector that is passed as the 3rd argument to the
`sections` method ("#results li") is used to find a number of capybara
elements. Each capybara element found using the css selector is used to
create a new instance of the `SearchResultSection` and becomes its root
element. So if the css selector finds 3 `li` elements, calling
`search_results` will return an array containing 3 instances of
`SearchResultSection`, each with one of the `li` elements as it's root
element.

#### 匿名セクション集合

You can define collections of anonymous sections the same way you would
define a single anonymous section:

```ruby
class SearchResults < SitePrism::Page
  sections :search_results, "#results li" do
    element :title, "a.title"
    element :blurb, "span.result-decription"
  end
end
```

#### セクションが存在するかテストする

Using the example above, it is possible to test for the existence of the
sections. As long as there is at least one section in the array, the
sections exist. The `sections` method adds a `has_<sections name>?`
method to the page/section that our section has been added to. Given the
following example:

```ruby
class SearchResultSection < SitePrism::Section
  element :title, "a.title"
  element :blurb, "span.result-decription"
end

class SearchResults < SitePrism::Page
  sections :search_results, SearchResultSection, "#results li"
end
```

... here's how to test for the existence of the section:

```ruby
@results_page = SearchResults.new
# ...
@results_page.has_search_results?
```

...which allows pretty tests:

```ruby
Then /^there are search results on the page$/ do
  expect(@results.page).to have_search_results
end
```

#### セクションが出現するのを待つ

The final method added by `sections` to the page/section we're adding
our sections to is `wait_for_<sections name>`. It will wait for
capybara's default wait time for there to be at least one instance of
the section in the array of sections. For example:

```ruby
class SearchResultSection < SitePrism::Section
  element :title, "a.title"
  element :blurb, "span.result-decription"
end

class SearchResults < SitePrism::Page
  sections :search_results, SearchResultSection, "#results li"
end
```

... here's how to wait for the section:

```ruby
@results_page = SearchResults.new
# ...
@results_page.wait_for_search_results
@results_page.wait_for_search_results(10) #=> waits for 10 seconds instead of the default capybara timeout
```

## Load Validations

Load validations enable common validations to be abstracted and performed on a Page or Section to determine
when it has finished loading and is ready for interaction in your tests.

For example, suppose you have a page which displays a 'Loading...' message while the body of
the page is loaded in the background.  Load validations can be used to ensure tests wait for the correct url
to be displayed and the loading message removed before trying to interact with with the page.

Other use cases include Sections which are displayed conditionally and may take time to become ready to
interact with, such as animated lightboxes.

### Load Validationsの使用
Load validations can be used in three constructs:

* Passing a block to `Page#load`
* Passing a block to `Loadable#when_loaded`
* Calling `Loadable#loaded?`

#### Page#load

`Page#load` メソッドにて, ページロード後、 `when_loaded` のコンテキストでブロックが実行される.  
See `when_loaded` documentation below for further details.

Example:

```ruby
# Load the page and then execute a block after all load validations pass:
my_page_instance.load do |page|
  page.do_something
end
```

#### Loadable#when_loaded

The `Loadable#when_loaded` method on a Loadable class instance will yield the instance of the class into a
block after all load validations have passed.

If any load validation fails, an error will be raised with the reason, if given, for the failure.

Example:

```ruby
# Execute a block after all load validations pass:
a_loadable_page_or_section.when_loaded do |loadable|
  loadable.do_something
end
```

#### Loadable#loaded?

You can explicitly run load validations on a Loadable via the `loaded?` method.
This method will execute all load validations on the object and return a boolean value.
In the event of a validation failure, a validation error can be accessed via the `load_error`
method on the object, if any error message was emitted by the failing validation.

Example:

```ruby
it 'loads the page' do
  some_page.load
  some_page.loaded?    #=> true if/when all load validations pass
  another_page.loaded? #=> false if any load validations fail
  another_page.load_error #=> A string error message if one was supplied by the failing load validation, or nil
end
```

### Load Validationsの定義
A load validation is a block which returns a boolean value when evaluated against an instance of the Loadable.

```ruby
class SomePage < SitePrism::Page
  element :foo_element, '.foo'
  load_validation { has_foo_element? }
end
```

ブロックは２つの要素を持つ配列とすることもできる。１つ目はboolean値を返す要素、２つ目はエラーメッセージとして定義する。バリデーションエラーのデバッグのためにエラーメッセージを定義することを強く推奨する。

boolean値がtrueであればエラーメッセージは無視される。
```ruby
class SomePage < SitePrism::Page
  element :foo_element, '.foo'
  load_validation { [has_foo_element?, 'did not have foo element!'] }
end
```

Load validations may be defined on `SitePrism::Page` and `SitePrism::Section` classes (herein referred
to as `Loadables`) and are evaluated against an instance of the class when executed.

### Load Validation Inheritance and Execution Order

Any number of load validations may be defined on a Loadable class and will be inherited by its subclasses.

Load validations are executed in the order that they are defined.  Inherited load validations are executed
from the top of the inheritance chain (e.g. `SitePrism::Page` or `SitePrism::Section`) to the bottom.

For example:

```ruby
class BasePage < SitePrism::Page
  element :loading_message, '.loader'

  load_validation do
    wait_for_loading_message(1)
    [ has_no_loading_message?(wait: 10), 'loading message was still displayed' ]
  end
end

class FooPage < BasePage
  set_url '/foo'

  section :form, '#form'
  element :some_other_element, '.myelement'

  load_validation { [has_form?, 'form did not appear'] }
  load_validation { [has_some_other_element?, 'some other element did not appear'] }
end
```

上の例では `loaded?`は `FooPage`として呼ばれ、バリデーションは次の順序で実行される:

1. The `SitePrism::Page` default load validation will check `displayed?`
2. The `BasePage` load validation will wait for the loading message to disappear.
3. The `FooPage` load validation will wait for the `form` element to be present.
4. The `FooPage` load validation will wait for the `some_other_element` element to be present.

NOTE: `SitePrism::Page` includes a default load validation on `page.displayed?` which is applied
to all pages.  It is therefore not necessary to define a load validation for this condition on
inheriting page objects.

## Capybara Query Optionsを使用
When querying an element, section or a collection of elements or sections, you may
supply Capybara query options as arguments to the element and section methods in order
to refine the results of the query and enable Capybara to wait for all of the conditions
necessary to properly fulfill your request.

Given the following sample page and elements:

```ruby
class SearchResultSection < SitePrism::Section
  element :title, "a.title"
  element :blurb, "span.result-decription"
end

class SearchResults < SitePrism::Page
  element :footer, ".footer"
  sections :search_results, SearchResultSection, "#results li"
end
```

Asserting the attributes of an element or section returned by any method may fail if
the page has not finished loading the element(s):

```ruby
@results_page = SearchResults.new
# ...
expect(@results_page.search_results.size).to == 25 # This may fail!
```

The above query can be rewritten to utilize the Capybara :count option when querying for
the collection, which in turn causes Capybara to expect some number of results to be returned.
The method calls below will succeed, provided the elements appear on the page within the timeout:

```ruby
@results_page = SearchResults.new
# ...
@results_page.has_search_results? :count => 25
# OR
@results_page.search_results :count => 25
# OR
@results_page.wait_for_search_results nil, :count => 25 # wait_for_<element_name> expects a timeout value to be passed as the first parameter or nil to use the default timeout value.
```

Now we can write pretty, non-failing tests without hard coding these options
into our page and section classes:

```ruby
Then /^there are search results on the page$/ do
  expect(@results.page).to have_search_results :count => 25
end
```

This is supported for all of the Capybara options including, but not limited to :count, :text,
:wait, etc. This can also be used when defining page objects. Eg:

```ruby
class SearchResults < SitePrism::Page
    element :footer, ".footer"
    element :view_more, "li", text: "View More"
    sections :search_results, SearchResultSection, "#results li"
end
```

### Capybara Optionsをサポートするメソッド
The following element methods allow Capybara options to be passed as arguments to the method:

```ruby
@results_page.<element_or_section_name> :text => "Welcome!"
@results_page.has_<element_or_section_name>? :count => 25
@results_page.has_no_<element_or_section_name>? :text => "Logout"
@results_page.wait_for_<element_or_section_name> :count => 25
@results_page.wait_until_<element_or_section_name>_visible :text => "Some ajaxy text appears!"
@results_page.wait_until_<element_or_section_name>_invisible :text => "Some ajaxy text disappears!"
```

## Test views with Page objects

It's possible to use the same page objects of integration tests for view tests, too,
just pass the rendered HTML to the ```load``` method:

```ruby
require 'spec_helper'

describe 'admin/things/index' do
  let(:list_page) { AdminThingsListPage.new }
  let(:thing) { build(:thing, some_attribute: 'some attribute') }

  it 'contains the things we expect' do
    assign(:things, [thing])

    render template: 'admin/things/index'

    list_page.load(rendered)

    expect(list_page.rows.first.some_attribute).to have_text('some attribute')
  end
end
```

## IFrames

SitePrism allows you to interact with iframes. An iframe is declared as
a `SitePrism::Page` class, and then referenced by the page or section it
is embedded into. Like a section, it is possible to test for the
existence of the iframe, wait for it to exist as well as interact with
the page it contains.

### iframeの作成
An iframe is declared in the same way as a Page:

```ruby
class MyIframe < SitePrism::Page
  element :some_text_field, "input.username"
end
```

To expose the iframe, reference it from another page or class using the `iframe`
method. The `iframe` method takes 3 arguments; the name by which you
would like to reference the iframe, the page class that represents the
iframe, and an ID or class by which you can locate the iframe. For example:

```ruby
class PageContainingIframe < SitePrism::Page
  iframe :my_iframe, MyIframe, "#my_iframe_id"
end
```

The third argument to the `iframe` method must
contain a selector that will locate the iframe node.

### iframeが存在するかテストする
Like an element or section, it is possible to test for an iframe's
existence using the auto-generated `has_<iframe_name>?` method. Using
the above example, here's how it's done:

```ruby
@page = PageContainingIframe.new
# ...
@page.has_my_iframe? #=> true
expect(@page).to have_my_iframe
```

### iframeを待つ
Like an element or section, it is possible to wait for an iframe to
exist by using the `wait_for_<iframe_name>` method. For example:

```ruby
@page = PageContainingIframe.new
# ...
@page.wait_for_my_iframe
```

### iframeコンテンツとの相互作用:

Since an iframe contains a fully fledged SitePrism::Page, you are able
to interact with the elements and sections defined within it. Due to
capybara internals it is necessary to pass a block to the iframe instead
of simply calling methods on it; the block argument is the
SitePrism::Page that represents the iframe's contents. For example:

```ruby
# SitePrism::Page representing the iframe
class Login < SitePrism::Page
  element :username, "input.username"
  element :password, "input.password"
end

# SitePrism::Page representing the page that contains the iframe
class Home < SitePrism::Page
  set_url "http://www.example.com"

  iframe :login_area, Login, "#login_and_registration"
end

# cucumber step that performs login
When /^I log in$/ do
  @home = Home.new
  @home.load

  @home.login_area do |frame|
    #`frame` is an instance of the `Login` class
    frame.username.set "admin"
    frame.password.set "p4ssword"
  end
end
```

## SitePrismの設定

SitePrism can be configured to change its behaviour.

### CapybaraのImplicit Waitsを使う

By default, SitePrism element and section methods do not utilize
Capybara's implicit wait methodology and will return immediately if
the element or section requested is not found on the page.  Add the
following code to your spec_helper file to enable Capybara's implicit
wait methodology to pass through:

```ruby
SitePrism.configure do |config|
  config.use_implicit_waits = true
end
```

This enables you to replace this:

```ruby
# wait_until methods always wait for the element to be present on the page:
@search_page.wait_for_search_results

# Element and section methods do not:
@search_page.search_results
```

with this:

```ruby
# With implicit waits enabled, use of wait_until methods is no longer required. This method will
# wait for the element to be found on the page until the Capybara default timeout is reached.
@search_page.search_results
```

## VCRでSitePrismを使う
There's a SitePrism plugin called `site_prism.vcr` that lets you use
SitePrism with the VCR gem. Check it out here:

* https://github.com/dnesteryuk/site_prism.vcr

# エピローグ
So, we've seen how to use SitePrism to put together page objects made up
of pages, elements, sections and iframes. But how to organise this stuff? There
are a few ways of saving yourself having to create instances of pages
all over the place. Here's an example of this common problem:

```ruby
@home = Home.new # <-- noise
@home.load
@home.search_field.set "Sausages"
@home.search_field.search_button.click
@results_page = SearchResults.new # <-- noise
expect(@results_page).to have_search_result_items
```

The annoyance (and, later, maintenance nightmare) is having to create
`@home` and `@results_page`. It would be better to not have to create
instances of pages all over your tests.

ページインスタンスを返すメソッドを含むクラスを作ることで、この問題に対処する
Eg:

```ruby
# our pages

class Home < SitePrism::Page
  #...
end

class SearchResults < SitePrism::Page
  #...
end

class Maps < SitePrism::Page
  #...
end

# here's the app class that represents our entire site:

class App
  def home
    Home.new
  end

  def results_page
    SearchResults.new
  end

  def maps
    Maps.new
  end
end

# and here's how to use it:

#first line of the test...
Given /^I start on the home page$/ do
  @app = App.new
  @app.home.load
end

When /^I search for Sausages$/ do
  @app.home.search_field.set "sausages"
  @app.home.search_button.click
end

Then /^I am on the results page$/ do
  expect(@app.results_page).to be_displayed
end

# etc...
```

The only thing that needs instantiating is the App class - from then on
pages don't need to be initialized, they are now returned by methods on
@app. Maintenance win!
