# Implement — oh-my-nixos framework scaffold

> 执行计划。每步结尾给验证命令。失败回到对应 prd/design 修订。所有写码在 `task.py start` 后再开始；此处只规划。

## 阶段划分

- **P0 骨架**：目录 + flakelight 接线，让 flake 能 eval（无 host）。
- **P1 host 框架**：mkHost/mkHome + option 注册 + presets 注册，example 不含实功能先能 eval 出 nixosConfiguration。
- **P2 系统层 modules**：core/security/persistence/disk 工厂逐模块 + impermanence 接。
- **P3 home 层**：core/persistence/graphic/terminal/presets，home-manager 集成进 example。
- **P4 集成 inputs**：impermanence / disko / lanzaboote / agenix / stylix / vscode-ext / nixpkgs-stable overlay 实接线。
- **P5 生成器**：`tools/generate` bash 脚本 + `apps.generate` 暴露。
- **P6 出箱验证**：example host `nixos-rebuild build` 与 disko `--mode destroy-and-format` dry-build。

## 有序清单

1. **[P0.1]** 建 `nix/{lib,modules/{core,security,persistence,disk,presets},home/{core,persistence,graphic,terminal,presets}}/` `hosts/example/` `homes/example/` `secrets/` `tools/generate/` `modules/` 目录；各放 `.gitkeep` 或 `README`。
   - 验证：`git status` 看到目录。
2. **[P0.2]** 写 `flake.nix`：保留现有 `inputs`，加 `flakelight ./. { inherit inputs; nixosModules = ...; }`；写 `nix/flake-module.nix`(flakelight module)作为框架装配点。
   - 验证：`nix flake check` 不报语法。
3. **[P1.1]** `nix/lib/mk-host.nix`：`mkHost = { name, modules, ... }: inputs.nixpkgs.lib.nixosSystem { modules = [ baseModules..  inputs.impermanence.nixosModules.impermanence  inputs.home-manager.nixosModules.home-manager  inputs.disko.nixosModules.disko  inputs.lanzaboote.nixosModules.lanzaboote  inputs.stylix.nixosModules.stylix  agenix  ] ++ modules; }`。
   - 验证：`nix eval .#nixosConfigurations.example.config.system.name`（先占位）。
4. **[P1.2]** `nix/presets/host.nix` 注册 §3.1 全部 option（mkOption）；同 `nix/home/presets/home.nix` 注册 §3.2。
   - 验证：`nix eval .#nixosConfigurations.example.config.oh-my-nixos.host.preset` 返回 `laptop`。
5. **[P1.3]** 遍历 `hosts/*/host.nix` 生成 nixosConfigurations 的逻辑装进 `nix/flake-module.nix`；写 `hosts/example/host.nix`（preset=laptop, homes=[example], disk 暂 null）
   - 验证：`nix eval .#nixosConfigurations.example.type` 成功。
6. **[P2.1]** `core/*`：nix settings / boot（systemd-boot，无 lanzaboote 先）/ locale / ssh（§6 默认）/ network（preset 分支）/ zram / firmware。各为独立 module，被 presets 引入。
   - 验证：`nixos-rebuild build --flake .#example`（VM 不必跑）编译通过。
7. **[P2.2]** `persistence/system.nix`：impermanence 系统层 + 白名单（§4.1）。
   - 验证：eval 出 `environment.persistence."/persist"`。
8. **[P2.3]** `disk/factory.nix` + 预设 `mkDisk`：实现 §4.2 的 ext4/btrfs × luks × impermanence 变体；example `disk.nix` 调一次。
   - 风险点：disko schema 易错，用 `disko –dry-run` 验证语法。
   - 验证：`nix run .#disko-deps` 不行则用 `nix eval .#nixosConfigurations.example.config.disko.devices` 不报错。
9. **[P2.4]** `security/{lanzaboote,luks,agenix}.nix`：各由 option 开关接线。
   - 验证：option 关时 module 为 no-op（eval diff 与 P2.1 一致）。
10. **[P2.5]** `presets/{laptop,desktop,server}.nix`：mkIf preset enum 设差异 config（D9）。
    - 验证：server preset 下 ` snd`/audio 关、`networkd`、无图形。
11. **[P3.1]** `home/core/{shell,git,cli,editor,fonts}.nix` + `home/persistence/default-whitelist.nix`（§5）。
    - 验证：`nix eval .#nixosConfigurations.example.config.home-manager.users.example.home.stateVersion`。
12. **[P3.2]** `home/persistence/{ephemeral,persist-dots,persistent}.nix` 三档实现。
    - 验证：切 `home.persistence` 三值，eza eval 三档都能构出。
13. **[P3.3]** `home/graphic/gnome.nix` 默认 + `kde/hyprland/sway`(opt-in 占位)；`home/terminal/`;`home/presets/{graphic,terminal}.nix`。
    - 验证：GNOME 装出 `services.xserver.desktopManager.gnome.enable`。
14. **[P4.1]** 接 stylix §4.5；nix-vscode-extensions 给 `programs.vscode` extensions；nixpkgs-stable overlay §7。
    - 验证：`nix eval .#nixosConfigurations.example.config.home-manager.users.example.programs.vscode`。
15. **[P4.2]** `secrets/secrets.nix` 公钥表 + example 占位 `example-wifi.age` + 文档 Brinde。
    - 验证：`agenix -e secrets/example-wifi.age` 可解（公钥为占位时文档说明）。
16. **[P5.1]** `tools/generate/generate.sh`：实现 §8 的交互与产出；校验 enum 值；落盘后提示 `nixos-rebuild` / `disko` 下一步。
    - 验证：跑一次生成器在临时目录产出 `hosts/<fake>/`，eval 成功。
17. **[P6.1]** 全量验证：`nix flake check`、`nixos-rebuild build --flake .#example`、`nix build .#nixosConfigurations.example.config.system.build.diskoScript --dry-run`、生成器一日志。
    - 验证：三条全绿。

## 验证命令汇总

```bash
nix flake check
nix eval .#nixosConfigurations.example.config.system.name
nixos-rebuild build --flake .#example
nix build .#nixosConfigurations.example.config.system.build.diskoScript --dry-run
nix run .#generate -- --dry-run   # 若生成器支持 dry-run
```

## 风险点 / 回滚点

- **disko schema 错**：用 `--dry-run` 早发现；错了回到 P2.3 重写工厂，不改其它层。
- **impermanence 白名单引发"用户丢数据"**：P3.1 完成后立刻手测白名单覆盖；清单缺失回 `nix/home/persistence/default-whitelist.nix` 补，不改 outside。
- **lanzaboote 复杂性**：默认关，opt-in 失败不影响主线；可暂留占位 module 先。
- **home-manager 跨 host override 约定打架**：P3.1 先定 `home-manager.users.<user>` 引用规则写进 design §4.3；冲突回 P1.3 改接线。
- **stylix base16 scheme 改名/弃用**：僵爬少，出现则换 `tokyonight` 同类 scheme。

## 实现前自检（`task.py start` 之前）

- [ ] prd 通过 PRD 收敛 pass。
- [ ] design.md 与 prd D1–D9 对齐无矛盾。
- [ ] implement.md 步骤可独立验证。
- [ ] implement.jsonl / check.jsonl 至少各 1 条真实 spec/research 条目（见下）。