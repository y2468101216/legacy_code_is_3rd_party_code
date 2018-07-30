# Legacy code is 3rd party code 

## 前言

本篇原文出處為 ： [Legacy code is 3rd party code](https://robertbasic.com/blog/legacy-code-is-3rd-party-code/)
已獲得原文作者同意翻譯，如有翻譯錯誤敬請賜教

## 專有名詞解釋

TDD : [測試驅動開發](https://zh.wikipedia.org/zh-tw/%E6%B5%8B%E8%AF%95%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91)
mock : 測試替身

## 本文

在 TDD 社群內有一個建議是這樣的：*我們永遠都不可以 mock 那些我們所不擁有的( library )*。我相信這是一個好建議且盡我所能的遵守這個原則。
當然有些人會說我們起初就不應該就用 mock。無論你在哪個 TDD (mock) 陣營，我想你都該知道在這條建議（我們永遠都不可以 mock 那些我們所不擁有的）之後還潛藏一個更好的建議。一個因為人們看到 mock 就抓狂而被遺忘的建議。

這個隱藏的建議是：*我們應當在我們的 code 與第三方套件之間建立 interfaces, clients, bridges, adapters*。這樣我們將可以在那些無關緊要的 interfaces 之上建立我們的 mock。這樣的話我們就可以藉由這些由我們所建立與使用的 interfaces 使我們的 code 與第三方套件解耦。
一個經典的例子就是在 php 世界裡我們經常使用 [guzzle](http://docs.guzzlephp.org/en/stable/) 作為我們建立與使用 http client 的 library，
而且我們都是直接注入 guzzle client 在我們的 application code 裡面。

為何舉這個例子的其中一個原因是因為 [guzzle](http://docs.guzzlephp.org/en/stable/) 大部分的時候都提供遠比你所需要的功能更多的功能。
建立一個自有的 http client 注入 [guzzle](http://docs.guzzlephp.org/en/stable/) 並只揭露那些你所需要的 api 可以限制開發者們能做的事情。
另一個原因為如果 [guzzle](http://docs.guzzlephp.org/en/stable/) 在未來改變了 api ，我們將只需要更改一個地方，而非修改 code 然後祈禱它不會弄壞任何東西。這是我使用 mock 兩個好原因（原文是 ：  Two very good reasons and I haven’t even mentioned mocks! 不太確定怎翻)

我認為這個建議並不會難以遵守，第三方套件存在於我們 application 的單獨目錄底下，一般而言是 vendor/ 或者 library/，他與我們有擁有不同的 namespace 跟 命名慣例。這是一個較簡單的方法去減少我們 code 對第三方套件的依賴。

### 如果我們也把這個原則放在 legacy code ?

如果我們將我們的 legacy code 與第三方套件一視同仁？如果 legacy code 只有維護，只修 bug 跟把螺絲鎖緊（where we only fix bugs and tweak bits and pieces of it 不太確定怎翻），這可能會很難做，又或者根本適得其反。但是一旦我們需要撰寫 new code 在正在使用的 legacy code， 我相信我們應當將 
legacy code 視為第三方套件，至少對 new code 是如此

如果可以盡可能的把 new code 與 legacy code 放在不同的目錄並使用不同的 namespace。這是一個可行的方案，畢竟我已經很久沒看到有系統不用 autoloading 。
但是除了直接把 new code 與 legacy code 直接連結起來，我們可以嘗試為 legacy code 建立 interface 並使用於 new code。

Legacy code 太常是全充滿了 *GOD OBJECT* - 一個方法做太多事（that do way too many things），legacy code 向 global state 伸出魔手，有 public properties 或者 magic methods 揭露 privates 就算他們本來就是就是 public，static method 讓我們可以在任何各種地方使用他們。但你猜猜，
導致這種爛情況的第一名就是這種方法。

另一個 - 或者是最重大的一個問題就是當我們將要準備改，修， hack 它時，並沒有把它視作第三方套件。當我們發現第三方套件有 bug 或者需要新功能時是如何做的？
我們是發 pull request 或者 issue。我們*不會*在 vendor/ 底下修改它，為何我們不將 legacy code 如此做，而是十指交扣祈禱它不會弄壞掉任何東西。

讓我們嘗試寫為 god object interface 裡面只揭露我們需要的功能，替代掉直接將 new code 寫在 legacy code 裡。所以我們在 legacy code 有一個知道任何人事 User object，它知道如何改密碼、改 email， 如何提昇論壇成員成為板主，如何更新 user 檔案、
更新通知設定，如何儲存他自己， etc。

src/Legacy/User.php:

```php
<?php
namespace Legacy;
class User
{
    public $email;
    public $password;
    public $role;
    public $name;

    public function promote($newRole)
    {
        $this->role = $newRole;
    }

    public function save()
    {
        db_layer::save($this);
    }
}
```

這是一個未加工的範例，但是給我們展示了幾個問題：每個 property 都是 public 這導致它易於修改
我們必須在呼叫 save method 以前清晰的記得這些改變。

讓我們限制並禁止我們自己直接使用這些 public properties 並不得不去推測 legacy system 是如何在任何時候都可以提高 user

src/LegacyBridge/Promoter.php:

```php
<?php
namespace LegacyBridge;
interface Promoter
{
    public function promoteTo(Role $role);
}
```

src/LegacyBridge/LegacyUserPromoter.php:

```php
namespace LegacyBridge;
class LegacyUserPromoter implements Promoter
{
    private $legacyUser;
    public function __construct(Legacy\User $user)
    {
        $this->legacyUser = $user;
    }

    public function promoteTo(Role $newRole)
    {
        $newRole = (string) $newRole;
        // I guess you thought $role in legacy is a string? Guess again!
        $legacyRoles = [
            Role::MODERATOR => 1,
            Role::MEMBER => 2,
        ];
        $newLegacyRole = $legacyRoles[$newRole];
        $this->legacyUser->promote($newLegacyRole);
        $this->legacyUser->save();
    }
}
```

現在每當我們要提昇 user 時，我們將使用我們的 new code *LegacyBridge\Promoter* interface，來與我們的 legacy system 進行所有 promote 細節 的互動。

### 改變 legacy 字詞

一個專門用於 legacy code 的 interface 讓我們有機會改善系統的設計。interface 可以讓我們避免在 legacy 的潛在命名錯誤。比如將一個 user's role 從板主降為一般成員並不是 *promotion* 而是 *demotion*，就算 legacy code 將兩件事視為一樣的，也補能阻止我們建立兩個 interface：

src/LegacyBridge/Promoter.php：

```php
<?php
namespace LegacyBridge;
interface Promoter
{
    public function promoteTo(Role $role);
}
```

src/LegacyBridge/LegacyUserPromoter.php:

```php
<?php
namespace LegacyBridge;
class LegacyUserPromoter implements Promoter
{
    private $legacyUser;
    public function __construct(Legacy\User $user)
    {
        $this->legacyUser = $user;
    }

    public function promoteTo(Role $newRole)
    {
        if ($newRole->isMember()) {
            throw new \Exception("Can't promote to a member.");
        }
        $legacyMemberRole = 2;
        $this->legacyUser->promote($legacyMemberRole);
        $this->legacyUser->save();
    }
}
```

src/LegacyBridge/Demoter.php:

```php
<?php
namespace LegacyBridge;
interface Demoter
{
    public function demoteTo(Role $role);
}
```

src/LegacyBridge/LegacyUserDemoter.php:

```php
<?php
namespace LegacyBridge;
class LegacyUserDemoter implements Demoter
{
    private $legacyUser;
    public function __construct(Legacy\User $user)
    {
        $this->legacyUser = $user;
    }

    public function demoteTo(Role $newRole)
    {
        if ($newRole->isModerator()) {
            throw new \Exception("Can't demote to a moderator.");
        }
        $legacyModeratorRole = 1;
        $this->legacyUser->promote($legacyModeratorRole);
        $this->legacyUser->save();
    }
}
```

這不是一個很大的改變，但卻使 code 看起來更清晰。

現在下次當你想要 legacy code 使用 method，試試看建立 interface。這可能並非可行，也可能是要付出昂貴的代價。
我知道 static method 在 god object 是真的十分簡單於使用且可以讓你的工作做快一點，但至少要記住這個選項。你
可能只須做出一點改變就可以改善系統的架構。

Happy hackin’!