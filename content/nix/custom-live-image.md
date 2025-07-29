---
date: "2025-07-30T00:04:14+03:00"
title: "Создание кастомного ISO образа для NixOS"
weight: 100
draft: true
---

В данной статье собраны готовые примеры позволяющие создать кастомизированный ISO образ с NixOS 
и home-manager, используя инструменты предоставляемые nixpkgs и nixos-generators.

Репозиторий с аналогичными примерами и готовым `justfile` для их сборки можно [найти тут](https://github.com/sysraccoon/nixos-custom-iso-examples)

Данный материал также доступен в видеоформате:
{{<youtube Rd7JIzm1SNc>}}

## 1 вариант без flakes и home-manager

В произвольном месте создаём файл `configuration.nix`:

```nix {filename="configuration.nix"}
{
  pkgs,
  modulesPath,
  ...
}:
{
  imports = [
    # Тут определены различные параметры которые нужны для 
    # создания загрузочного образа NixOS
    "${modulesPath}/installer/cd-dvd/installation-cd-minimal.nix"
  ];

  # Изменяем архитектуру если ожидается другая
  nixpkgs.hostPlatform = "x86_64-linux";
  # Подставляем версию на основе которой будут браться 
  # значения по умолчанию для различных параметров.
  system.stateVersion = "24.05"; 

  # Ниже можно определить любые другие параметры как в обычном configuration.nix
  environment.systemPackages = with pkgs; [
    git
    neovim
  ];
}
```

Для сборки можно использовать `nix-build`:

```sh
nix-build "<nixpkgs/nixos>" \
  -A config.system.build.isoImage -I nixos-config=configuration.nix
```

Или как альтернативный вариант `nixos-generators`:

```sh
nix run nixpkgs#nixos-generators -- \
  --format iso --configuration configuration.nix -o result
```

В любом случае собранный ISO образ можно найти в директории `result`.

## 2 вариант с flakes но без home-manager

Файл `configuration.nix` остаётся таким же как и в предыдущем пункте, добавляется только `flake.nix`
(и автоматически генерируемый `flake.lock`):

```nix {filename="flake.nix"}
{
  inputs = {
    # Версию можно поменять при необходимости
    nixpkgs.url = "github:nixos/nixpkgs/nixos-24.05";
  };

  outputs = { nixpkgs, ... }: {
    # Вместо liveImage также можно подставить любое другое удобное название
    nixosConfigurations.liveImage = nixpkgs.lib.nixosSystem {
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

Сборка варианта с флейком осуществляется следующим образом:

```sh
# если название конфигурации (liveImage) был изменён ранее в конфигурации
# то его также необходимо изменить тут ↓
nix build .#nixosConfigurations.liveImage.config.system.build.isoImage
```

Или в качестве альтернативного варианта можно всё также использовать `nixos-generators` передавая ему параметр `--flake`:
```sh
nix run nixpkgs#nixos-generators -- --format iso --flake .#liveImage -o result
```

## 3 вариант без flakes но с home-manager

Создаём файл `home.nix` с следующим содержимым:

```nix {filename="home.nix"}
{ pkgs, ... }:
{
  imports = [
    <home-manager/nixos>
  ];
  home-manager = let
    # по умолчанию в installation-cd-minimal.nix определён один пользователь 
    # с именем "nixos", изменение данной переменной потребует также 
    # создание соответствующего пользователя в configuration.nix
    username = "nixos";
  in {
    # Про useGlobalPkgs и useUserPackages описано дальше по тексту
    useGlobalPkgs = true;
    useUserPackages = true;
    users.${username} = { pkgs, osConfig, ... }: {
      # Системную конфигурацию можно получить как аргумент osConfig и брать 
      # из неё свойства при необходимости
      home.stateVersion = osConfig.system.stateVersion;
      home.username = username;
      home.homeDirectory = "/home/${username}";

      # Остальные параметры аналогичны обычному конфигу home-manager
      home.packages = with pkgs; [
        jq
        fzf
        curl
        # ...
      ];
    };
  };
}
```

`useGlobalPkgs` определяет нужно ли использовать раздельные настройки для 
nixpkgs в home-manager и системной конфигурации, если выставлено в `true`, 
то будет использоваться тот же самый pkgs что и в системном конфиге, если 
в `false`, то отдельный pkgs, который можно настраивать внутри конфига 
home-manager. 

`useUserPackages` определяет какой путь будет использоваться для пакетов 
установленных через home.packages. Если выставлено в `true`, то задействуется 
`/etc/profiles`, иначе `$HOME/.nix-profile`.

Дальше `home.nix` нужно импортировать в `configuration.nix` (он описан в первом варианте):
```nix {filename="configuration.nix"}
{
  pkgs,
  modulesPath,
  ...
}:
{
  imports = [
    # Оставляем как есть
    "${modulesPath}/installer/cd-dvd/installation-cd-minimal.nix"
    # Добавляем файл с конфигурацией для home-manager
    ./home.nix
  ];
  # Остальные параметры приведённые ранее
}
```

Сборка ничем не отличается от первого варианта:

```sh
nix-build "<nixpkgs/nixos>" \
  -A config.system.build.isoImage -I nixos-config=configuration.nix
```

С использованием `nixos-generators`:

```sh
nix run nixpkgs#nixos-generators -- \
  --format iso --configuration configuration.nix -o result
```

## 4 вариант с flakes и home-manager

Такой вариант требует внесения изменений как в `flake.nix`, так и в `home.nix`. Начнём с `flake.nix`:
```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-24.05";

    # Добавляется сам home-manager в inputs, версия при этом 
    # должна быть равна nixpkgs
    home-manager = {
      url = "github:nix-community/home-manager/release-24.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = {
    nixpkgs,
    home-manager,
    ...
  }: let
  in {
    nixosConfigurations.liveImage = nixpkgs.lib.nixosSystem {
      # таким образом можно прокинуть home-manager в качестве 
      # параметра во все модули
      specialArgs = { inherit home-manager; };
      modules = [
        # но при желании можно импортировать его прямо тут
        # home-manager.nixosModules.home-manager
        ./configuration.nix
      ];
    };
  };
}
```

В `home.nix` вместо `<home-manager/nixos>` используем передаваемый в качестве параметра модуль home-manager:

```nix {filename="home.nix"}
{ 
  pkgs, 
  home-manager, # получаем как параметр
  ... 
}:
{
  imports = [
    # изменили модуль
    home-manager.nixosModules.home-manager   
  ];

  # Остальное без изменений
}
```

Сборка осуществляется также как во втором варианте:

```sh
nix build .#nixosConfigurations.liveImage.config.system.build.isoImage
```

С `nixos-generators`:
```sh
nix run nixpkgs#nixos-generators -- --format iso --flake .#liveImage -o result
```

## Дополнительные ресурсы

- nixos-generators github: [link](https://github.com/nix-community/nixos-generators)
- nixpkgs installation modules: [link](https://github.com/NixOS/nixpkgs/tree/master/nixos/modules/installer/cd-dvd)
- nixos manual (building ISO image section): [link](https://nixos.org/manual/nixos/stable/#sec-building-image)
- vimjoyer youtube video: [link](https://www.youtube.com/watch?v=-G8mN6HJSZE&ab_channel=Vimjoyer)
