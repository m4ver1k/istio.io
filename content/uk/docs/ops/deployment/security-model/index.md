---
title: Модель безпеки
description: Описує модель безпеки Istio.
weight: 10
owner: istio/wg-security-maintainers
test: n/a
---

Цей документ має на меті описати стан безпеки різних компонентів Istio і те, як можливі атаки можуть вплинути на систему.

## Компоненти {#components}

Istio постачається з різноманітними додатковими компонентами, які будуть розглянуті тут. Для загального огляду перегляньте розділ [Архітектура Istio](/docs/ops/deployment/architecture/). Зазначимо, що розгортання Istio є надзвичайно гнучким; нижче ми здебільшого розглядатимемо найгірші сценарії.

### Istiod {#istiod}

Istiod служить основним компонентом панелі управління Istio, часто виконуючи роль [компонента XDS](/docs/concepts/traffic-management/) та виступаючи як [Центр Сертифікації mТLS](/docs/concepts/security/).

Istiod вважається високопривілейованим компонентом, схожим на сервер API Kubernetes.

* Він має високі привілеї Kubernetes RBAC, зазвичай включаючи доступ для читання `Secret` та запис через webhook.
* Виконуючи роль CA, він може надавати різні сертифікати.
* Виконуючи роль панелі управління XDS, він може налаштовувати проксі для виконання довільних дій.

Таким чином, безпека кластера тісно повʼязана з безпекою Istiod. Дотримання [найкращих практик безпеки Kubernetes](https://kubernetes.io/docs/concepts/security/) щодо доступу до Istiod є вкрай важливим.

### Втулок Istio CNI {#istio-cni-plugin}

Istio може бути розгорнуто з [втулком Istio CNI як DaemonSet](/docs/setup/additional-setup/cni/). Цей `DaemonSet` відповідає за налаштування мережевих правил в Istio, забезпечуючи прозоре перенаправлення трафіку за потреби. Це альтернатива контейнеру `istio-init`, про який [нижче](#sidecar-proxies).

Оскільки CNI `DaemonSet` змінює мережеві правила на вузлі, він потребує підвищеного `securityContext`. Однак, на відміну від [Istiod](#istiod), це локальний привілей вузла. Наслідки цього дивись [нижче](#node-compromise).

Оскільки це обʼєднує підвищені привілеї, необхідні для налаштування мережі, в одному podʼі, а не в *всіх* podʼах, цей параметр зазвичай є рекомендованим.

### Проксі sidecar {#sidecar-proxies}

Istio може [опціонально](/docs/overview/dataplane-modes/) розгортати проксі sidecar поруч із застосунком.

Проксі sidecar потребує налаштування мережі для направлення всього трафіку через проксі. Це можна зробити за допомогою [втулка Istio CNI](#istio-cni-plugin) або розгортання `initContainer` (`istio-init`) у podʼі (це відбувається автоматично, якщо втулок CNI не розгорнуто). Контейнер `istio-init` потребує прав `NET_ADMIN` та `NET_RAW`. Однак ці права діють лише під час ініціалізації — основний контейнер sidecar є повністю непривілейованим.

Додатково, проксі sidecar не потребує жодних привілеїв Kubernetes RBAC.

Кожен проксі sidecar має право запитувати сертифікат для повʼязаного Службового облікового запису Podʼа.

### Gateways та Waypoints {#gateways-and-waypoints}

{{< gloss "gateway" >}}Шлюзи{{< /gloss >}} та {{< gloss "waypoint">}}Waypoints{{< /gloss >}} функціонують як автономні розгортання проксі. На відміну від [sidecar](#sidecar-proxies), вони не потребують жодних змін у мережі та, відповідно, не потребують привілеїв.

Ці компоненти працюють з власними службовими обліковими записами, відмінними від ідентифікації застосунків.

### Ztunnel {#ztunnel}

{{< gloss "ztunnel" >}}Ztunnel{{< /gloss >}} виступає як проксі на рівні вузла. Віг вимагає прав `NET_ADMIN`, `SYS_ADMIN` та `NET_RAW`. Як і [втулок Istio CNI](#istio-cni-plugin), він є **локальними привілеями для вузла**. Ztunnel не має жодних повʼязаних привілеїв Kubernetes RBAC.

Ztunnel має право запитувати сертифікати для будь-яких Службових облікових записів podʼів, що працюють на тому ж вузлі. Аналогічно до [kubelet](https://kubernetes.io/docs/reference/access-authn-authz/node/), він явно не дозволяє запитувати довільні сертифікати. Це, знову ж таки, забезпечує, що ці привілеї є **локальними для вузла**.

## Властивості перехоплення трафіку {#traffic-capture-properties}

Коли pod стає частиною мережі, весь вхідний TCP трафік буде перенаправлено до проксі. Це включає як mTLS/{{< gloss >}}HBONE{{< /gloss >}}, так і незашифрований трафік. Будь-які придатні [політики](/docs/tasks/security/authorization/) для робочого навантаження будуть застосовані перед перенаправленням трафіку до робочого навантаження.

Однак, Istio наразі не гарантує, що *вихідний* трафік буде перенаправлено до проксі. Дивіться [обмеження перехоплення трафіку](/docs/ops/best-practices/security/#understand-traffic-capture-limitations). Таким чином, необхідно дотримуватись кроків з [забезпечення безпеки вихідного трафіку](/docs/ops/best-practices/security/#securing-egress-traffic), якщо потрібні політики для вихідного трафіку.

## Властивості mTLS Properties {#mutual-tls-properties}

[mTLS](/docs/concepts/security/#mutual-tls-authentication) забезпечує основу для більшості безпекових позицій Istio. Нижче пояснено різні властивості, які забезпечує mTLS для безпеки Istio.

### Центр Сертифікації {#certificate-authority}

Istio має вбудований Центр Сертифікації.

Стандартно, CA дозволяє автентифікувати клієнтів на основі одного з наступних варіантів:

* Токен JWT Kubernetes з аудиторією `istio-ca`, який перевіряється за допомогою `TokenReview` Kubernetes. Це стандартний метод в podʼах Kubernetes.
* Наявний сертифікат mTLS.
* JWT-токени користувачів, перевірені з використанням OIDC (вимагає налаштування).

CA видаватиме лише ті сертифікати, які запитуються для ідентифікаторів, для яких клієнт пройшов автентифікацію.

Istio також може інтегруватися з різними сторонніми CA; будь ласка, зверніться до документації з безпеки відповідних CA для детальнішої інформації.

### Клієнт мTLS {#client-mtls}

{{< tabset category-name="dataplane" >}}
{{< tab name="Режим sidecar" category-value="sidecar" >}}
У режимі sidecar клієнтський sidecar буде [автоматично використовувати TLS](/docs/ops/configuration/traffic-management/tls-configuration/#auto-mtls) під час підключення до сервісу, який підтримує mTLS. Це також можна [явно налаштувати](/docs/ops/configuration/traffic-management/tls-configuration/#sidecars). Зверніть увагу, що це автоматичне виявлення залежить від того, як Istio асоціює трафік з Service. [Непідтримувані типи трафіку](/docs/ops/configuration/traffic-management/traffic-routing/#unmatched-traffic) або [обмеження конфігурації](/docs/ops/configuration/mesh/configuration-scoping/) можуть завадити цьому.

Під час [підключення до бекенду](/docs/concepts/security/#secure-naming) набір дозволених ідентифікацій обчислюється на рівні Service, базуючись на обʼєднанні всіх ідентифікацій бекенду.
{{< /tab >}}

{{< tab name="Режим ambient" category-value="ambient" >}}
У режимі ambient Istio автоматично використовуватиме mTLS під час підключення до будь-якого бекенду, що підтримує mTLS, і перевірятиме, чи відповідає ідентифікація пункту призначення очікуваній ідентифікації для виконання робочого навантаження.

Ці властивості відрізняються від режиму sidecar тим, що є властивостями окремих робочих навантажень, а не сервісу. Це дозволяє виконувати більш точні перевірки автентифікації та підтримувати ширший спектр робочих навантажень.
{{< /tab >}}
{{< /tabset >}}

### Сервер mTLS {#server-mtls}

Стандартно Istio приймає mTLS та не mTLS трафік (що часто називають «дозвільним режимом»). Користувачі можуть вибрати суворий режим, написавши правила `PeerAuthentication` або `AuthorizationPolicy`, що вимагають mTLS.

При встановленні зʼєднання mTLS перевіряється сертифікат учасника. Крім того, перевіряється належність ідентифікаторів до одного домену довіри. Щоб дозволити перевірку лише певних ідентифікаторів, можна використовувати `AuthorizationPolicy`.

## Досліджені типи компрометації {#compromise-types-explored}

На основі наведеного вище огляду ми розглянемо вплив на кластер, якщо різні частини системи будуть скомпрометовані.
У реальному світі існує безліч різних змінних навколо будь-якої атаки на безпеку:

* Наскільки легко її виконати
* Які попередні привілеї потрібні
* Як часто вона може бути використана
* Які наслідки (повне віддалене виконання, відмова в обслуговуванні і т.д.).

У цьому документі ми розглянемо найгірший сценарій: скомпрометований компонент означає, що зловмисник має повну можливість віддаленого виконання коду.

### Компрометація робочого навантаження {#workload-compromise}

У цьому випадку відбувається компрометація робочого навантаження застосунку (pod).

Pod [*може* мати доступ](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting) до токена службового облікового запису. Якщо це так, компрометація навантаження може поширитися від одного pod, компрометуючи весь службовий обліковий запис.

{{< tabset category-name="dataplane" >}}
{{< tab name="Режим sidecar" category-value="sidecar" >}}
У моделі sidecar проксі розташований поруч із pod і працює в межах тієї ж зони довіри. Скомпрометований застосунок може втручатися у роботу проксі через адмін-API або інші інтерфейси, зокрема для вилучення матеріалу приватного ключа, дозволяючи іншому агенту видавати себе за робоче навантаження. Припускається, що компрометація робочого навантаження включає також компрометацію проксі sidecar.

Таким чином, скомпрометоване навантаження може:

* Надсилати довільний трафік, з mTLS або без нього. Воно може обійти будь-яку конфігурацію проксі або навіть сам проксі. Зазначимо, що Istio не пропонує політики авторизації для вихідного трафіку, тому обхід політики авторизації вихідного трафіку не відбувається.
* Приймати трафік, який вже був призначений для застосунку. Воно може обійти політики, налаштовані в проксі sidecar.

Основний висновок тут полягає в тому, що, хоча скомпрометоване навантаження може поводитися шкідливо, це не дає йому змоги обійти політики в *інших* навантаженнях.
{{< /tab >}}

{{< tab name="Режим ambient" category-value="ambient" >}}
У режимі ambient проксі вузла не розміщено в межах podʼа і він працює в іншій зоні довіри як частина окремого podʼа.

Скомпрометований застосунок може надсилати довільний трафік. Однак він не контролює проксі вузла, який визначатиме, як обробляти вхідний і вихідний трафік.

Крім того, оскільки pod не має доступу до токена службового облікового запису для запиту сертифіката mTLS, можливості для латерального переміщення зменшуються.
{{< /tab >}}
{{< /tabset >}}

Istio пропонує різноманітні функції, які можуть обмежити наслідки такої компрометації:

* [Функції спостережуваності](/docs/tasks/observability/) можуть бути використані для ідентифікації атаки.
* [Політики](/docs/tasks/security/authorization/) можуть обмежити типи трафіку, які навантаження може надсилати або отримувати.

### Компрометація проксі - Sidecars {#proxy-compromise---sidecars}

У цьому випадку проксі sidecar є скомпрометованим. Оскільки sidecar і застосунок знаходяться в одній зоні довіри, це функціонально еквівалентно до [компрометації робочого навантаження](#workload-compromise).

### Компрометація проксі - Waypoint {#proxy-compromise---waypoint}

У цьому випадку скомпрометовано [проксі waypoint](#gateways-and-waypoints). Хоча у waypoint немає жодних привілеїв для використання зловмисником, він обслуговує (потенційно) багато різних сервісів та робочих навантажень. Скомпрометований waypoint може отримувати, переглядати, змінювати або відхиляти весь трафік для цих навантажень.

Istio пропонує можливість [налаштування гранулярності розгортання waypoint](/docs/ambient/usage/waypoint/#useawaypoint). Користувачі можуть розглянути можливість розгортання більш ізольованих waypoint для посилення ізоляції.

Оскільки waypoint працює з окремою ідентичністю від застосунків, які він обслуговує, компрометація waypoint не означає, що можна видати себе за застосунки користувача.

### Компрометація проксі - Ztunnel {#proxy-compromise---ztunnel}

У цьому випадку компрометовано проксі [ztunnel](#ztunnel).

Скомпрометований ztunnel надає зловмиснику контроль над мережею вузла.

Ztunnel має доступ до матеріалу приватних ключів для кожного застосунку, який працює на його вузлі. Компрометація ztunnel може призвести до їхнього отримання та використання в інших місцях. Однак латеральний рух до ідентичностей за межами розміщених робочих навантажень неможливий; кожен ztunnel має авторизацію доступу лише до сертифікатів для навантажень, що працюють на його вузлі, обмежуючи радіус ураження скомпрометованого ztunnel.

### Компрометація вузла {#node-compromise}

У цьому випадку скомпрометовано вузол Kubernetes. І [Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/node/), і Istio спроєктовані для обмеження масштабу шкоди від компрометації одного вузла, так, щоб компрометація одного вузла не призводила до [компрометації всього кластера](#cluster-api-server-compromise).

Проте атака має повний контроль над будь-якими робочими навантаженнями, що виконуються на цьому вузлі. Наприклад, вона може скомпрометувати будь-які спільно розміщені [waypoint](#proxy-compromise---waypoint), локальний [ztunnel](#proxy-compromise---ztunnel), будь-які [sidecar](#proxy-compromise---sidecars), будь-які спільно розміщені [екземпляри Istiod](#istiod-compromise) тощо.

### Компрометація кластера (API Server) {#cluster-api-server-compromise}

Компрометація API-сервера Kubernetes фактично означає, що весь кластер і mesh скомпрометовані. На відміну від більшості інших векторів атаки, Istio мало що може зробити для обмеження масштабу такої атаки. Скомпрометований API-сервер надає зловмиснику повний контроль над кластером, включно з можливістю виконувати `kubectl exec` на довільних podʼах, видаляти будь-які `AuthorizationPolicies` Istio або навіть повністю видаляти Istio.

### Компрометація Istiod {#istiod-compromise}

Компрометація Istiod зазвичай призводить до такого ж результату, як і [компрометація API-сервера](#cluster-api-server-compromise). Istiod є дуже привілейованим компонентом, який слід ретельно захищати. Дотримання [кращих практик безпеки](/docs/ops/best-practices/security) є важливим для підтримки захисту кластера.