# Laravel 的事件概念及實作方式

在 Laravel 中事件 (Event) 是一種發佈-訂閱模式(Publish-Subscribe)的實作方式
，它可以使應用程式中的邏輯從執行中分離出來，讓整個程式碼更加清晰明瞭。
當一個特定的事件發生時，會發送出該事件並且觸發訂閱這個事件的所有處理函式。

透過 `event()` 函式或 `Event facade` 來發送事件。例如 event(new UserRegistered($user)); 或 Event::dispatch(new UserRegistered($user));
就能觸發 UserRegistered 事件，並將 $user 物件傳入事件實例中。

要訂閱一個事件，需要先建立一個 `Listener` 類別來實作訂閱事件的邏輯。
像是 UserRegistered 事件就要建立一個 UserRegisteredListener 類別，
在裡面有一個 handle 函式可以處理 UserRegistered 事件。

之後在 EventServiceProvider 類別中註冊這個 Listener，系統啟動時自動載入。

註冊方式可以透過 protected $listen 屬性來註冊，這個屬性是一個陣列，陣列的 key 是事件類別名稱， value 則是對應的 Listener 類別名稱 。

也可以透過 listen 方法來註冊， listen 的第一個參數是事件類別名稱，第二個參數是對應的 Listener 類別名稱。
這些註冊過程都是在 EventServiceProvider 類別中處理

---

listen 方法有第三個參數用來指定監聽者的優先級，讓一些監聽器在其他監聽器之前執行。還有 subscribe 方法可以註冊一個類別中`所有事件監聽器`。

如果你需要在一個類別中註冊多個事件監聽器，則可以使用 subscribe 方法，而不需要在 EventServiceProvider 類別中個別註冊每個監聽器。

最後在需要的地方使用 event() 函式或 Event facade 來發送事件，監聽器就會被執行

而使用 `Queue` 來處理事件可以提高效能，因為排入佇列後事件就不需要同步處理，由Queue worker 負責處理，可以大幅降低主程式等待事件處理的時間。

#php #php/laravel #php/laravel/events #listener #eventserviceprovider #publish-subscribe #queue #backend