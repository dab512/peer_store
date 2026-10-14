# PeerStore — macOS installation and bootstrap

Публичное распространение бинарников и резервный discovery. Исходники закрыты и здесь не публикуются.

## Начать на macOS

Apple Silicon (arm64) и Intel (amd64), preview `v0.1.0-preview.1`:

```bash
/bin/bash -c 'set -euo pipefail; umask 077; d=$(mktemp -d); trap '"'"'rm -rf "$d"'"'"' EXIT; trap '"'"'exit 130'"'"' INT; trap '"'"'exit 143'"'"' TERM; curl --proto '"'"'=https'"'"' --proto-redir '"'"'=https'"'"' --tlsv1.2 --fail --location --silent --show-error --output "$d/install-macos.sh" https://github.com/dab512/peer_store/releases/download/v0.1.0-preview.1/install-macos.sh; printf "%s  %s\n" cfa3b85d23ac9716744dfcfb672d0b39a5a0915b0e6c594c03e3bf75eb7a0fca "$d/install-macos.sh" > "$d/checksum"; shasum -a 256 -c "$d/checksum"; /bin/bash "$d/install-macos.sh"'
```

Команда скачивает установщик фиксированного релиза, проверяет его закреплённый SHA-256 **до выполнения**, затем установщик проверяет SHA-256 подходящего бинарного архива. Устанавливает в `~/.peerstore/bin/peerstore`, создаёт локальную identity в `~/.peerstore/node`; sudo, изменения firewall/роутера и PATH не нужны.

**Новый узел получит `awaiting_admission`.** Его публичная подписанная заявка — `~/.peerstore/node/config/join-request.json`; private key остаётся на компьютере. Установка не даёт права участия в mesh; dynamic join пока не реализован. Существующие ключи не перезаписываются. Запуск неизвестного ключа не обходит эту проверку.

[Релиз](https://github.com/dab512/peer_store/releases/tag/v0.1.0-preview.1) содержит архивы двух архитектур, `install-macos.sh`, `SHA256SUMS` и public `nodes.json`. Hash установщика: `cfa3b85d23ac9716744dfcfb672d0b39a5a0915b0e6c594c03e3bf75eb7a0fca`. Независимо доверенный hash этой команды — начальная точка доверия; чтение hash и скрипта из одного подменённого источника не обеспечивает независимую проверку.

## Сеть и список узлов

- Network ID: `7396bb9d407371c52e6eb2f7a70245cd6805b36730f7a70847521ceb00a7fc98`.
- Root public-key fingerprint (SHA-256): `e20adaad1eb32c01eace87531c2ca055689f380205c36a60f3f6246aa4252a74`.
- [Подписанный пополняемый список](bootstrap/nodes.json), raw: https://raw.githubusercontent.com/dab512/peer_store/main/bootstrap/nodes.json.
- Первый узел — `dab512-mac`, peer ID `12D3KooWGPNmYDqK7qKP4UZ39UEswV5xttmb5q3RWRw7RG9U1MMi`, статус **awaiting_address**, public endpoints пока отсутствуют. Он проверен локально на TCP 4001; external listener не включён без разрешения владельца.

PeerStore спрашивает известных допущенных peers; GitHub — fallback, свежий подписанный cache — offline fallback. Signature, network/root binding, expiry, revision/rollback/equivocation проверяются при любом источнике. Адреса в списке — routing hints, не membership и не замена trust root. Чтобы добавить запись, владелец сети подписывает новую ревизию, затем публикует nodes.json; private signing keys сюда не кладутся.

Обновление списка на запущенном подготовленном узле:

```bash
~/.peerstore/bin/peerstore bootstrap refresh --instance-dir ~/.peerstore/node
```

Допуск/настройка уже существующей identity требует отдельного подтверждения владельца сети. После допуска prepared узел запускается так:

```bash
~/.peerstore/bin/peerstore node --instance-dir ~/.peerstore/node
```

## Узнать свой адрес и создать ссылку

На **запущенном допущенном** узле:

```bash
~/.peerstore/bin/peerstore address --instance-dir ~/.peerstore/node
~/.peerstore/bin/peerstore link --instance-dir ~/.peerstore/node --version <version-id> --self-bootstrap
```

Кандидаты interfaces/listener отличаются от verified addresses. Известный observer выполняет новое обратное TCP соединение и Noise authentication ожидаемого peer-id. Без observer результат incomplete; loopback/LAN не объявляются public internet reachability. `--observer <full-multiaddr>` задаёт конкретный известный peer. `--allow-local-address` разрешает local-only self-bootstrap для тестов.

## Границы preview

Все прикладные mesh requests/replies подписаны; admission gate допускает только разрешённые node keys. Подпись привязана к сети, sender/recipient/service, payload, nonce и expiry; unsigned RPC fallback отсутствует. Admin API — только private Unix socket. На открытый TCP-порт могут приходить неизвестные packets/handshakes: scanning/DoS криптография не исключает.

Первый Mac — один validator, без отказоустойчивости нескольких независимых машин. Текущий consensus trust статический и истекает через 24h после genesis; signed directory его не продлевает. Dynamic membership, DHT, relay/NAT traversal, private-store encryption и updater ещё не завершены; этот preview не является полноценным internet mesh rollout. Связанность глубины 1–4 и история графика пока заданы в спеках, не реализованы в runtime.
