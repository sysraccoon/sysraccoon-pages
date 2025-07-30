---
date: "2025-07-30T20:29:29+03:00"
draft: true
title: "Введение в Nix Flakes"
weight: 90
tags: draft
---

В коммьюнити Nix-а можно часто увидеть упоминание Flake, но по моему опыту у новичков возникают различные трудности с восприятием того что это и зачем оно нужно. В данной статье попытаюсь дать вам основы которые помогут с ними освоиться.

## Что такое Flake?

Nix Flake - это формат описания Nix-проекта, который включает:

- `flake.nix` - описывающий зависимости и результаты сборки на языке Nix
- `flake.lock` - автоматически генерируемый файл содержащий точные версии всех зависимостей

Команды вроде `nix build`, `nix develop`, `nixos-rebuild switch` умеют работать с этим форматом получая из него нужные данные.

{{< callout type="warning" >}}
  Nix Flakes являются экспериментальной функцией и в будущем могут произойти серьёзные изменения. Данный материал актуален на июль 2025 (nixos 25.05, nix 2.28.4).

  Для работы Nix Flakes необходимо добавить следующие параметры в `/etc/nixos/configuration.nix` если используется NixOS:
  ```nix {filename="/etc/nixos/configuration.nix"}
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
  ```

  Или в `/etc/nix/nix.conf` если вы используете Nix на **любом другом дистрибутиве**:
  ```ini {filename="/etc/nix/nix.conf"}
  experimental-features = nix-command flakes
  ```
{{< /callout >}}

## Про flake.nix

Чтобы начать использовать данный формат достаточно в корне проекта создать файл с именем `flake.nix`, в котором есть `inputs` и `outputs`:

```nix {filename="flake.nix"}
{
  inputs = {
    # ...
  };

  outputs = { ... }: {
    # ...
  };
}
```

В `inputs` описываются зависимости которые нужны для сборки данного проекта:
```nix {filename="flake.nix",hl_lines=[3]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
  };

  outputs = { ... }: {
    # ...
  };
}
```

В данном случае мы пытаемся получить доступ к содержимому nixpkgs в ветке `nixos-25.05` ([github](https://github.com/NixOS/nixpkgs/tree/nixos-25.05)). При желании можно указать сразу несколько разных версий nixpkgs:

```nix {filename="flake.nix",hl_lines=[4]}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
    nixpkgs-unstable.url = "github:nixos/nixpkgs/nixos-unstable";
  };

  outputs = { ... }: {
    # ...
  };
}
```

`outputs` это функция принимающая загруженные и распакованные зависимости из `inputs` и возвращающая результат работы (пакеты, окружения, конфиги и т.д):

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";
    nixpkgs-unstable.url = "github:nixos/nixpkgs/nixos-unstable";
  };

  outputs = { nixpkgs, nixpkgs-unstable, ... }: {
    # Через packages определяются пакеты
    packages.x86_64-linux.default = /*...*/; # Пакет
    # Через devShells преднастроенные dev-окружения
    devShells.x86_64-linux.default = /*...*/; # 
    # Через nixosConfigurations системные конфигурации для NixOS
    nixosConfigurations.hostname = /*...*/;
  };
}
```

Разными результатами пользуются разные команды Nix-а. Например, при попытке вызвать `nix build` будет задействован `packages.<system>.default`, где `<system>` это архитектура системы на которой запущен Nix. При вызове `nix develop` будет создано окружение в соответствии с `devShells.<system>.default`. Можно также явно задать название необходимого объекта, который также будет искаться в соответствии с типом команды и текущей архитектурой `nix build .#default`.

Подробный список того что может находится в `outputs` можно посмотреть в [NixOS Wiki](https://nixos.wiki/wiki/Flakes#Output_schema) или под спойлером ниже.

{{<details title="Output Schema" closed="true">}}

- `<system>` используемая архитектура, например `x86_64-linux`, `aarch64-linux`, `i686-linux`, `x86_64-darwin`
- `<name>` название атрибута, например `hello`.
- `<flake>` путь к проекту где располагается `flake.nix`.
- `<store-path>` это путь в `/nix/store/...`

```nix {filename="flake.nix"}
{
  outputs = { self, ... } @ inputs: {
    # Используется в `nix flake check`
    checks."<system>"."<name>" = derivation;
    # Используется в `nix build .#<name>`
    packages."<system>"."<name>" = derivation;
    # Используется в `nix build .`
    packages."<system>".default = derivation;
    # Используется в `nix run .#<name>`
    apps."<system>"."<name>" = {
      type = "app";
      program = "<store-path>";
    };
    # Используется в `nix run . -- <args?>`
    apps."<system>".default = { type = "app"; program = "..."; };
    # Форматтеры (alejandra, nixfmt или nixpkgs-fmt)
    formatter."<system>" = derivation;
    # Используется в nixpkgs, также доступен через `nix build .#<name>`
    legacyPackages."<system>"."<name>" = derivation;
    # Оверлеи доступные для других флейков
    overlays."<name>" = final: prev: { };
    # Оверлей по умолчанию
    overlays.default = final: prev: { };
    # NixOS модули используемые другими флейками
    nixosModules."<name>" = { config, ... }: { options = {}; config = {}; };
    # Модуль по умолчанию
    nixosModules.default = { config, ... }: { options = {}; config = {}; };
    # Используется в `nixos-rebuild switch --flake .#<hostname>`
    # nixosConfigurations."<hostname>".config.system.build.toplevel должен быть деривацией
    nixosConfigurations."<hostname>" = {};
    # Используется в `nix develop .#<name>`
    devShells."<system>"."<name>" = derivation;
    # Используется в `nix develop`
    devShells."<system>".default = derivation;
    # Задачи сборки для Hydra
    hydraJobs."<attr>"."<system>" = derivation;
    # Используется в `nix flake init -t <flake>#<name>`
    templates."<name>" = {
      path = "<store-path>";
      description = "Описание шаблона";
    };
    # Используется в `nix flake init -t <flake>`
    templates.default = { path = "<store-path>"; description = ""; };
  }
}

```

Есть также дополнительные варианты используемые некоторыми другими инструментами, как пример [home-manager](https://github.com/nix-community/home-manager) может использовать конфигурацию определяемую через `homeConfigurations.<name>`

{{</details>}}

## Про flake.lock

Как упоминалось ранее, помимо `flake.nix` есть также файл `flake.lock`. Он выполняет ту же функцию что `package-lock.json` в npm (JS) или `uv.lock` в uv (Python), а именно фиксирует точные версии завимостей.

`flake.lock` создаётся автоматически при первой попытке использовать `flake.nix` через какую-либо команду (например `nix build`) и представляет собой json такого вида:

```json {filename="flake.lock"}
{
  "nodes": {
    "nixpkgs": {
      "locked": {
        "lastModified": 1753749649,
        "narHash": "sha256-+jkEZxs7bfOKfBIk430K+tK9IvXlwzqQQnppC2ZKFj4=",
        "owner": "nixos",
        "repo": "nixpkgs",
        "rev": "1f08a4df998e21f4e8be8fb6fbf61d11a1a5076a",
        "type": "github"
      },
      "original": {
        "owner": "nixos",
        "ref": "nixos-25.05",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "nixpkgs-unstable": {
      "locked": {
        "lastModified": 1753694789,
        "narHash": "sha256-cKgvtz6fKuK1Xr5LQW/zOUiAC0oSQoA9nOISB0pJZqM=",
        "owner": "nixos",
        "repo": "nixpkgs",
        "rev": "dc9637876d0dcc8c9e5e22986b857632effeb727",
        "type": "github"
      },
      "original": {
        "owner": "nixos",
        "ref": "nixos-unstable",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "root": {
      "inputs": {
        "nixpkgs": "nixpkgs",
        "nixpkgs-unstable": "nixpkgs-unstable"
      }
    }
  },
  "root": "root",
  "version": 7
}
```

В дальнейшем изменение в `flake.lock` происходят только если изменяются `inputs` в `flake.nix` или при явном вызове `nix flake update` что подтягивает новые версии.

Всё это позволяет увеличить вероятность что проект успешно собираемый в одной системе, соберётся в другой.

