# ArmaReforger-reverse-auth
Попытка разобраться, как работает авторизация в сервисе Bohemia, для игры Arma Reforger

## Основы
1. Для начала нам требуется понять, каким образом происходит передача пакетов(https/tcp/udp/wss)
2. Далее начинаем анализировать пакеты и ищем пакет авторизации
3. После того как мы нашли нужный пакет авторизации начинаем анализ тела запроса и выносим вердикт

## Глава 1 - Начало. Поиск пакетов
Здесь я вам расскажу о том, как разработчики сделали умный ход и попытались скрыть пакеты от проксирования.
Начнём с того, как лучше всего стоит анализировать трафик и ответом естественно будет `Wireshark`, но мне повезло и я с помощью `x64dbg` смог заранее предсказать что используется `WEB REST API` и сразу перешёл к попытке проксирования с помощью программы `Charles`.
Но как только я начал проксировать пакеты я столкнулся с проблемой, пакетов игры то нет:

<img src="https://github.com/user-attachments/assets/e9550040-7f5f-4975-aee4-3a547d2a3b7c" width="500" />

### Переходим сразу к тому, как я решил эту проблему:
1. Изначально стоило попытаться понять, почему так происходит, и ответ не заставил себя ждать, разработчики просто отключили поддержку прокси. И что же тогда делать? Читаем далее...
2. Так как обычно практически все игровые движки используют `cURL`, и я не так давно провёл своё исследование в этой области, то всё стало куда проще, нужно было просто перехватить функции `cURL`
   
### CURL или как достаточно просто перехватить пакеты приложения
Для начала разберёмся что такое `сURL`, как нам говорит _ЯНДЕКС_: Curl — это утилита командной строки для передачи данных с сервера или на сервер. Но если мы комнём глубже, мы узнаем что существует ещё и `LIBCURL` и что далее?
+ Ищем нужную функцию(от `cUrl`) в игре которая отвечает за передачу данных для создания цельного пакета
+ Смотрим как устанавливаются параметры **PROXY** (гуглим)
+ И просто устанавливаем их самостоятельно

Этой нужной волшебной функцией является: `curl_easy_setopt(...)`, и всё что нам остаётся это перехватить функцию и сделать примерно так:
```cpp
CURLcode __stdcall curl_easy_setopt_hook(CURL* handle, CURLoption option, ...){
    if (option == CURLOPT_URL) {
        original_curl_easy_setopt(handle, CURLOPT_PROXY, (void*)"127.0.0.1:8888");
        original_curl_easy_setopt(handle, CURLOPT_PROXY_SSL_VERIFYPEER, (void*)0L);
    }
}
```
Лично я собрал всё это в DLL и заинжектил в процесс, и вот что получилось:
<img src="https://github.com/user-attachments/assets/69c9b869-f331-44f1-9834-0509d585507f" width="500" />

Это победа, так что переходим к анализу авторизации
## Глава 2 - Знакомый токен. Анализ авторизации
Чтож, теперь приступаем к интересному, а точнее, как всё же работает авторизация.
Начинаем смотреть пакет и видим основную информацию:
1. url: https://api-ar-id.bistudio.com/game-identity/api/v1.1/identities/reforger/auth?include=profile
2. тип запроса: post
3. Тело запроса: 
```json
{
	"platform": "steam",
	"token": "FAAAADFFe1SXFGECFC8RDwEAEAGXqKxmGAAAAAEAAAAFAAAACqQ+...",
	"platformOpts": {
		"appId": "1874880"
	}
}
```
Теперь переходим к попытке понять, а что за токен то такой?
Изначально я предпологал что это, что то самописное, ибо никогда не видел подобного токена, но меня смутило вот это: `FAAAA` в начале токена.
Я сразу же полез смотреть логи игры через консоль `Steam` и заметил что вызываются функции:
```
0030680 ArmaReforgerSteam.exe:2883590 > IClientUser::GetAuthTicketForWebApi( NULL, ) = 11, 
00033894 ArmaReforgerSteam.exe:2883590 > IClientUser::CancelAuthTicket( 11, )
00033894 ArmaReforgerSteam.exe:2883590 > IClientUser::GetAuthTicketForWebApi( NULL, ) = 12,
```
Я сразу же полез писать небольшой скрипт, чтобы посмотреть что же там возвращает эта функция и я получил такой вывод: `1400000012B6CA4AFE06DF69142F110F010010019`.
Я растроился, ведь вывод не был таким, каким я его ожидал, пока мой друг не заметил что изначальный токен был похож на base64, я решил проверить теорию и через любой веб сервис декодировал base64 -> hex и ответ не заставил себя ждать:
`FAAAADFFe1SXFGECFC8RDwEAEAGXqKxmGAAAAAEAAAAFAAAACqQ... ----> 1400000031457b5497146102142f110f0...` и стало очевидно что это был просто закодированный тикет

## Глава 3 - А для чего это всё?. Вывод
Теперь мы подводим итог, и я обьясню для чего было это мини исследование. Изначально я эмулировал всё `API` игры, чтобы найти какие нибудь уязвимости, но столкнулся с проблемой что я мог получить доступ ко всему кроме `Workshop` который предоставляла игра.
Идея была такова, написать мини стим приложение которое будет взаимодействовать с моим `API`. Мой веб сервис, должен был запросить у моего приложение токен для авторизации в `Workshop`, а само приложение просто вызывала функцию получения токена, отправляла реквест с авторизацией и возвращала `accessToken`. Вот такое вот забавное исследование

## Внимание
Вся приведенная информация была добыта с целью изучения и не более, не надо мешать разработчикам творить игры
