# "NFT Marketplace" Audit.




## Critical:



### C1:

#### titile: 
Несанкционированный доступ к функции claim

#### code_line: 
42

#### description: 
Функцию claim может вызывать кто угодно, с любым аргументом “user”, это может привести к тому, что кто-то другой может вывести юзеру его награды в нежелательный для него момент. 

#### solution:
Рекомендую удалить аргумент “user”, а его вхождения заменить на “msg.sender”


### C2:

#### titile: 
В payRewards в качестве аргумента reward поступает обнуленный объект

#### code_line: 
50, 47

#### description: 
В функцию payRewards на 50 строчке поступает объект хранящийся в store, в момент вызова он копируется в memory, но перед его копированием этот же объект удаляется при вызове withdrawLastDeposit в 49 строчке, где он на 66 строчке удаляет этот объект(делает amount и timestamp равными нулю). В итоге в payRewards аргумент reward всегда равен = {amount: 0, timestamp: 0}, что приводит к неправильным расчетам.

#### solution:
Необходимо поменять тип хранения переменной reward в 47 строчке на “memory”, тогда независимо от удаления данных из store, в payRewards будет поступать сохраненная копия.


### C3:

#### titile: 
Ошибка в формуле расчета userReward в функции payRewards

#### code_line: 
59

#### description: 
В формуле 2 ошибки:

1. Ввиду отсутствия чисел с плавающей точкой в Solidity принято в первую очередь выполнять все операции умножения, и только потом деление, в противном случае теряется точность при расчетах.

2. (random % daysDelta) - здесь может быть деление на 0, если прошло менее 1 дня

#### solution:

1. 59 строчку надо отредактировать к виду

```solidity
uint256 userReward = reward.amount * (random % daysDelta) / PCT_DENOMINATOR;
```

2. Можно привести формулу к виду 

```solidity
uint256 userReward = daysDelta == 0 ? 0 : reward.amount * (random % daysDelta) / PCT_DENOMINATOR;
```


### C4:

#### titile: 
Функция buy не может отправить NFT покупателю

#### code_line: 
141

#### description: 
При выставлении на продажу NFT токен остается у владельца, что делает невозможным сделать перевод в 141 строчке, потому что перед вызовом “transferFrom” не от имени владельца, необходимо сделать аппрув токена на адрес получателя, но заранее не зная адрес получателя - нельзя сделать на него аппрув.


#### solution:

Для исправления нужно отредактировать функции “buy”, “discardFromSale” и “setForSale”, сделав так что бы, при выставлении на продажу NFT переводился на адрес маркетплейса(тем самым блокируя его), при продаже его надо отправлять уже с адреса контракта а не первоначального владельца, а так же вернуть NFT владельцу в случае отмены продажи.
Так же нужно реализовать контракте Marketplace интерфейс  IERC721Receiver, что бы ваш контракт был совместим с токенами ERC721 и мог их получать.


### C5:

#### titile: 
Архитектура функции “setForSale” нарушает принцип единственной ответственности, что может приводить к ошибкам во время работы.

#### code_line: 
109

#### description: 
Функция setForSale выполняет сразу 2 разных логических действия:

1. Создает новую продажу

2. Обновляет price и startTime у существующей

Такая логика не допустима (см C4)

#### solution:
Необходимо декомпозировать эту функцию на 2:

1. createForSale - которая будет создавать новую продажу, так же она должна выбрасывать ошибку в случае если токен уже выставлен на продажу, и если price == 0

2. updateForSale - которая будет обновлять данные продажи. Обновление startTime можно из нее исключить, так как этим занимается функция postponeSale.

Так же ввиду этих изменений, нужно отредактировать получение владельца токена, которая встречается на строчках 115, 121, 131: вместо “NFT_TOKEN.ownerOf(tokenId)” надо написать “items[tokenId].seller” 



## Medium:




### M1:

#### titile: 
Неэффективное хранение PAYMENT_TOKEN

#### code_line: 
29

#### description: 
Store-переменная PAYMENT_TOKEN устанавливается только в конструкторе, и больше нигде не перезаписывается. 

#### solution:
Ее можно объявить как “immutable” - тем самым она не будет занимать место в хранилище и сэкономит газ на ее чтении.


### M2:

#### titile: 
Неэффективное хранение REWARD_TOKEN

#### code_line: 
30

#### description: 
Store-переменная REWARD_TOKEN устанавливается только в конструкторе, и больше нигде не перезаписывается.

#### solution:
Ее можно объявить как “immutable” - тем самым она не будет занимать место в хранилище и сэкономит газ на ее чтении.


### M3:

#### titile: 
Неэффективное чтение данных из переменной _rewards в функции Rewardable.claim().

#### code_line: 
43, 47, 49, 50

#### description: 
Store-переменная _rewards многократно считывается в функции claim, что приводит к неэффективному расходованию газа.

#### solution:
Лучше в начале функции объявить переменную 

```solidity
Reward[] memory userRewards = _rewards[user];
```

и заменить вхождения “_rewards[user]” в строчках 43 и 47 на “userRewards”.

Так же в 47 строчке надо поменять тип хранения переменной reward на “memory”, что приведет к экономии газа на строчках 49 и 50. 


### M4:

#### titile: 
Неэффективная функция withdrawLastDeposit.

#### code_line: 
65 - 70

#### description: 
В целом это функция не эффективна по газу: в 66 строчке нет необходимости, ее заменяет 53 строчка, в остальном - withdrawLastDeposit вызывается в цикле, что приводит к многократной записи store-переменной _rewardsAmount в 68 строчке и трансфера PAYMENT_TOKEN в 69

#### solution:
Эффективнее ее удалить, и отредактировать функцию claim:

1. Добавить перед циклом в 46 строчке переменную “uint256 withdrawAmount;” 

2. Заменить код в 49 строчке на “withdrawAmount += reward.amount;”, 

3. После цикла, после 51 строчки добавить 2 строки кода: “_rewardsAmount -= withdrawAmount;” и “PAYMENT_TOKEN.transfer(user, withdrawAmount);”

В итоге должно получиться примерно это

```solidity
function claim(address user) external {
    uint256 length = _rewards[user].length;
    if (length == 0) revert NothingForClaim();

    uint256 withdrawAmount; // <- NEW

    for (uint256 i = 0; i < length; i++) {
        Reward storage reward = _rewards[user][length - i];

        withdrawAmount += reward.amount; // <- NEW
        payRewards(user, reward);
    }

    _rewardsAmount -= withdrawAmount; // <- NEW
    PAYMENT_TOKEN.transfer(user, withdrawAmount); // <- NEW

    delete _rewards[user];
}
```


### M5:

#### titile: 
Избыточный вызов REWARD_TOKEN.rewardUser() в цикле

#### code_line: 
61, 50

#### description: 
Отправка вознаграждений в функции payRewards на 61 строчке происходит в цикле на 50 строчке

#### solution:
Оптимальным решением будет заменить функцию payRewards на calculateRewards

```solidity
function calculateRewards(Reward memory reward) internal view returns(uint256 userReward) { // <- New
    uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, SEED)));
    uint256 daysDelta = (block.timestamp - reward.timestamp) / 1 days;
    userReward = reward.amount / PCT_DENOMINATOR * (random % daysDelta); // <- New
}
```

А функцию claim отредактировать к виду

```solidity
function claim(address user) external {
    uint256 length = _rewards[user].length;
    if (length == 0) revert NothingForClaim();

    uint256 rewardAmount; // <- New

    for (uint256 i = 0; i < length; i++) {
        Reward storage reward = _rewards[user][length - i];

        withdrawLastDeposit(user, reward.amount);
        rewardAmount += calculateRewards(reward); // <- New
    }

    if (rewardAmount > 0) { // <- New
        REWARD_TOKEN.rewardUser(user, rewardAmount); // <- New
    } // <- New
    delete _rewards[user];
}
```

Что сведет многократные вызовы REWARD_TOKEN.rewardUser() к единственному.


### M6:

#### titile: 
Неэффективное хранение NFT_TOKEN.

#### code_line: 
93

#### description: 
Store-переменная NFT_TOKEN устанавливается только в конструкторе, и больше нигде не перезаписывается

#### solution:
Ее можно объявить как “immutable” - тем самым она не будет занимать место в хранилище и сэкономит газ на ее чтении.


### M7:

#### titile: 
Неэффективное чтение данных из переменной items в функции Marketplace.buy().

#### code_line: 
134, 136, 137, 138, 140

#### description: 
Store-переменная items многократно считывается в функции buy, что приводит к неэффективному расходованию газа.

#### solution:
Лучше перед 134 строчкой объявить переменную 

```solidity
ItemSale memory itemSale = items[tokenId];
```

И заменить все вхождения “items[tokenId]” на “itemSale” в строчках 134, 136, 137, 138, 140



### M8:

#### titile: 
Неэффективная и сложная для понимания логика в функции postponeSale

#### code_line: 
123-127

#### description: 
Функция postponeSale откладывает начало продажи nft, но очень сложным и неэффективным способом

#### solution:
Строчки с 123 по 127 можно заменить на

```solidity
items[tokenId].startTime += postponeSeconds;
```

Это даст тот же результат, но сэкономит газ и сделает код более читаемым.


### M9:

#### titile: 
Переусложненные проверки в функции buy

#### code_line: 
136-138

#### description: 
В основном здесь проверяется существует ли продажа, а адрес владельца и так проверяется в 132 строчке

#### solution:
Лучшим решением здесь будет удалить строчки с 136 по 138, и в самом начале функции добавить проверку 

```solidity
if(items[tokenId].price == 0) revert InvalidSale();
```

Этого будет достаточно, при условии что нельзя создать продажу с нулевой ценой, следовательно у любого tokenId который не выставлен на продажу цена price всегда будет равен 0.


## Low:



### L1:

#### titile: 
Стиль именования переменной _rewardsAmount

#### code_line: 
32, 68, 74

#### description: 
Store-переменная _rewardsAmount объявлена как “public”, но в тоже время ее название начинается с “_” - такой стиль именования является общепринятым для локальных или приватных переменных, но не для публичных
 
#### solution:
Что бы код соответствовал общепринятым стайлгайдам рекомендую переименовать ее к “rewardsAmount”, и так же переименовать ее вхождения в строках 68 и 74


### L2:

#### titile: 
Предсказуемая генерация случайных чисел в payRewards

#### code_line: 
57

#### description: 
 Генерация случайности основанная на block.timestamp не является надежной, так как это значение подвержено манипуляциям со стороны майнеров, а так же мотивирует пользователей дожидаться максимально эффективного момента для вызова. Константа SEED здесь бесполезна, она видна пользователям как и все данные storage, независимо от модификаторов доступа “private”. 

#### solution:
Если нужна надежная генерация случайности предлагаю посмотреть в сторону решений которые предлагает ChainLink


### L3:

#### titile: 
Дублирование кода проверки владельца токена

#### code_line: 
115, 121

#### warning:
Зависит от C5, приведенные варианты решений являются примерными и должны быть отредактированы в соответствии с C5

#### description: 
Код проверяющий что вызывающий юзер является владельцем токена повторяется несколько раз

#### solution:
Можно провести рефакторинг вынеся это логику в подобный модификатор

```solidity
modifier isTokenOwner(uint256 tokenId) {
    if (NFT_TOKEN.ownerOf(tokenId) != msg.sender) revert NotItemOwner();
    _;
}
```

После чего добавить этот модификатор к функциям discardFromSale(114), postponeSale(120), и удалив из этих функций строку с проверкой (115, 121 соответственно)

Например так 

```solidity
function discardFromSale(uint256 tokenId) external isTokenOwner(tokenId) {
    delete items[tokenId];
}
```


## Notes:



### N1:
Перед деплоем контрактов, тщательно проведите аудит внешних контрактов “nftToken”, “paymentToken” и “rewardToken”. В них могут скрываться эксплойты, которые могут привести к взлому ваших контрактов. Используете только те внешние контракты, которым полностью доверяете.
