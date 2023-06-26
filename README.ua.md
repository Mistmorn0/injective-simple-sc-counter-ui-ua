# Приклад Injective Counter Контракту

Цей репо демонструє приклад того, як реалізувати підключення та взаємодію зі смарт-контрактом, розгорнутим на Injective Chain за допомогою модуля `injective-ts`, використовуючи Nuxt і Next.

Більше про `injective-ts` тут: [injective-ts wiki](https://github.com/InjectiveLabs/injective-ts/wiki)

Посилання на репозиторій смарт-контрактів: [cw-counter](https://github.com/InjectiveLabs/cw-counter)

У цьому README наведено приклад того, як реалізувати підключення та взаємодію зі смарт-контрактом у VanillaJS.
Ви можете знайти README для прикладів Nuxt і Next.js в папках відповідно.

## 1. Підготовка

Почніть зі встановлення залежностей модулів вузла, які ви збираєтеся використовувати (наприклад `@injectivelabs/sdk-ts` тощо...)

Ми можемо побачити модулі, які використовуються у цьому прикладі, у файлі `package.json`

Перш ніж ми почнемо використовувати модулі `@injectivelabs`, нам потрібно налаштувати наш bundler, додавши деякі плагіни, такі як `node-modules-polyfill`.

## 2. Налаштування сервісів

Далі нам потрібно налаштувати сервіси, які ми будемо використовувати.
Для взаємодії зі смарт-контрактом ми будемо використовувати  `ChainGrpcWasmApi` з `@injectivelabs/sdk-ts`.
Також нам знадобляться кінцеві точки мережі, які ми будемо використовувати (Mainnet або Testnet), які можна знайти в `@injectivelabs/networks`

Приклад:

```js
import { ChainGrpcWasmApi } from "@injectivelabs/sdk-ts";
import { Network, getNetworkEndpoints } from "@injectivelabs/networks";

export const NETWORK = Network.TestnetK8s;
export const ENDPOINTS = getNetworkEndpoints(NETWORK);

export const chainGrpcWasmApi = new ChainGrpcWasmApi(ENDPOINTS.grpc);
```

## 3. Wallet Strategy та Broadcast

Далі нам потрібно налаштувати клієнт Wallet Strategy та Broadcasting, імпортувавши `WalletStrategy` та `MsgBroadcaster` з `@injectivelabs/wallet-ts`.

Основна мета `@injectivelabs/wallet-ts` - запропонувати розробникам спосіб мати різні варіанти використання гаманців на Injective. Всі ці впровадження гаманців мають однаковий інтерфейс `ConcreteStrategy`, що означає, що користувачі можуть просто використовувати ці методи без необхідності знати базову концепцію конкретного гаманця, оскільки вони абстраговані.

Для початку вам потрібно створити екземпляр класу WalletStrategy, який дає вам можливість використовувати різні гаманці "з коробки". Ви можете змінити поточний гаманець, який використовується, за допомогою методу setWallet в екземплярі walletStrategy.
За замовчуванням використовується `Metamask`.

```js
import { WalletStrategy } from "@injectivelabs/wallet-ts";
import { Web3Exception } from "@injectivelabs/exceptions";

// Ці імпортовані дані з .env
import {
  CHAIN_ID,
  ETHEREUM_CHAIN_ID,
  IS_TESTNET,
  alchemyRpcEndpoint,
  alchemyWsRpcEndpoint,
} from "/constants";

export const walletStrategy = new WalletStrategy({
  chainId: CHAIN_ID,
  ethereumOptions: {
    ethereumChainId: ETHEREUM_CHAIN_ID,
    wsRpcUrl: alchemyWsRpcEndpoint,
    rpcUrl: alchemyRpcEndpoint,
  },
});
```

Щоб отримати адреси з гаманця, ми можемо скористатися наступною функцією:

```js
export const getAddresses = async (): Promise<string[]> => {
  const addresses = await walletStrategy.getAddresses();

  if (addresses.length === 0) {
    throw new Web3Exception(
      new Error("There are no addresses linked in this wallet.")
    );
  }

  return addresses;
};
```

Коли ми запускаємо цю функцію, вона відкриває ваш гаманець (за замовчуванням: Metamask), щоб ми могли підключитися, і ми можемо зберегти значення, що повертається, у змінній для подальшого використання.
Зверніть увагу, що ця функція повертає нам також і набір адрес Ethereum, які ми можемо перетворити в Injective адреси за допомогою утиліти `getInjectiveAddress`.

```js
import {getInjectiveAddress} from '@injectivelabs/sdk-ts'

const [address] = await getAddresses();
cosnt injectiveAddress = getInjectiveAddress(getInjectiveAddress)

```

Нам також знадобиться `walletStrategy` для `BroadcastClient`, включаючи використовувані мережі, які ми визначили раніше.

```js
import { Network } from "@injectivelabs/networks";
export const NETWORK = Network.TestnetK8s;

export const msgBroadcastClient = new MsgBroadcaster({
  walletStrategy,
  network: NETWORK,
});
```

## 4. Запит до Смарт-контракту

Тепер все налаштовано і ми можемо взаємодіяти зі смарт-контрактом.

Ми почнемо з запиту до смарт-контракту, щоб отримати поточний результат за допомогою сервісу `chainGrpcWasmApi`, який ми створили раніше,
і виклику функції `get_count` в смарт-контракті.

```js

      const response = (await chainGrpcWasmApi.fetchSmartContractState(
        COUNTER_CONTRACT_ADDRESS, // Адреса контракту
        toBase64({ get_count: {} }) // Нам потрібно перетворити наш запит в Base64
      )) as { data: string };

      const { count } = fromBase64(response.data) as { count: number }; // нам потрібно перетворити відповідь з Base64

      console.log(count)
```

## 5. Модифікація стану

Далі ми змінимо стан `count`.
Ми можемо зробити це, відправивши повідомлення ланцюжку за допомогою `Broadcast Client`, який ми створили раніше, і `MsgExecuteContractCompat` з `@injectivelabs/sdk-ts`.

Смарт-контракт, який ми використовуємо для цього прикладу, має 2 способи зміни стану:

- `increment`
- `reset`

Команда `increment` збільшує count на 1, а команда `reset` встановлює count на задане значення.
Зверніть увагу, що `reset` можна викликати тільки якщо ви є розробником смарт-контракту.

Коли ми викликаємо ці функції, наш гаманець відкривається для підписання повідомлення/транзакції і надсилає його в мережу.

```js
// Підготовка повідомлення

const msg = MsgExecuteContractCompat.fromJSON({
  contractAddress: COUNTER_CONTRACT_ADDRESS,
  sender: injectiveAddress,
  msg: {
    increment: {},
  },
});

// Підписання та відправка повідомлення в мережу

await msgBroadcastClient.broadcast({
  msgs: msg,
  injectiveAddress: injectiveAddress,
});
```

```js
// Підготовка повідомлення

const msg = MsgExecuteContractCompat.fromJSON({
  contractAddress: COUNTER_CONTRACT_ADDRESS,
  sender: injectiveAddress,
  msg: {
    reset: {
      count: parseInt(number, 10),
    },
  },
});

// Підписання та відправка повідомлення в мережу

await msgBroadcastClient.broadcast({
  msgs: msg,
  injectiveAddress: injectiveAddress,
});
```

## 6. Повний приклад

Нижче наведено приклад усього, про що ми дізналися до цього часу, зібраного воєдино.

```js
import { ChainGrpcWasmApi, getInjectiveAddress } from "@injectivelabs/sdk-ts";
import { Network, getNetworkEndpoints } from "@injectivelabs/networks";
import { WalletStrategy } from "@injectivelabs/wallet-ts";
import { Web3Exception } from "@injectivelabs/exceptions";

// Ці імпортовані дані з .env
import {
  CHAIN_ID,
  ETHEREUM_CHAIN_ID,
  IS_TESTNET,
  alchemyRpcEndpoint,
  alchemyWsRpcEndpoint,
} from "/constants";

const NETWORK = Network.TestnetK8s;
const ENDPOINTS = getNetworkEndpoints(NETWORK);

const chainGrpcWasmApi = new ChainGrpcWasmApi(ENDPOINTS.grpc);

const walletStrategy = new WalletStrategy({
  chainId: CHAIN_ID,
  ethereumOptions: {
    ethereumChainId: ETHEREUM_CHAIN_ID,
    wsRpcUrl: alchemyWsRpcEndpoint,
    rpcUrl: alchemyRpcEndpoint,
  },
});

export const getAddresses = async (): Promise<string[]> => {
  const addresses = await walletStrategy.getAddresses();

  if (addresses.length === 0) {
    throw new Web3Exception(
      new Error("There are no addresses linked in this wallet.")
    );
  }

  return addresses;
};

const msgBroadcastClient = new MsgBroadcaster({
  walletStrategy,
  network: NETWORK,
});

const [address] = await getAddresses();
const injectiveAddress = getInjectiveAddress(getInjectiveAddress);

async function fetchCount() {
  const response = (await chainGrpcWasmApi.fetchSmartContractState(
    COUNTER_CONTRACT_ADDRESS, // Адреса контракту
      toBase64({ get_count: {} }) // Нам потрібно перетворити наш запит в Base64
    )) as { data: string };

  const { count } = fromBase64(response.data) as { count: number }; // нам потрібно перетворити відповідь з Base64

  console.log(count)
}

async function increment(){
    const msg = MsgExecuteContractCompat.fromJSON({
    contractAddress: COUNTER_CONTRACT_ADDRESS,
    sender: injectiveAddress,
    msg: {
        increment: {},
        },
    });

    // Підписання та відправка повідомлення в мережу

    await msgBroadcastClient.broadcast({
        msgs: msg,
        injectiveAddress: injectiveAddress,
    });
}

async function main() {
    await fetchCount() // це буде лог: {count: 5}
    await increment() // Це відкриває ваш гаманець для підписання транзакції та її подальшої відправки в мережу
    await fetchCount() // тепер count дорівнює 6. log: {count: 6}
}

main()

```
