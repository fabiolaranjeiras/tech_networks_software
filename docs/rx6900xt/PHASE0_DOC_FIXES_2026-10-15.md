# Phase 0 — documentation fixes for the RX 6900 XT records (2026-10-15)

**For:** the executor agent that will follow `TEST_PLAN_LINUX_2026-10-15.md`. **Apply these before any test.**
**Why a patch list and not direct edits by the steward:** (a) the Drive connector available to the steward cannot edit file contents in place — every edit means re-emitting the whole file and creating a new Drive file ID; (b) other agents edited several of these files today (BIOS 3636, PSU details, a "logs organised" paragraph), so a blind overwrite would erase their work; (c) local edits on the synced folder cost nothing.

## Rules
1. Work on the local, Drive-synced folder `...\My Drive\Site\home_server\RTX_Ryzen9\`. **Re-read each target right before editing**; match by the quoted text. If the quoted text is not found verbatim (someone edited it, or markdown escaping differs), find the nearest equivalent sentence, apply the same intent, and write what you did in the summary. Do not guess silently.
2. Edit in place with a text-edit tool. **Do not rewrite whole files, do not change line endings** (files from Windows are CRLF), do not create copies, do not touch `_ARQUIVO*` or `My Drive\IT\backup\`.
3. Facts only. Keep `[medido]` / `[relatado]` / `[hipótese]` / `[não testado]` markers. A correction **replaces** the wrong text; never leave the wrong text beside the right one.
4. When finished, add the "Histórico" lines (item D-12) and list in your handoff what was applied, what was not found, and anything surprising.
5. **Duplicates:** on 2026-10-15 the Drive contained duplicate copies (same name, different IDs) of `AMD_Radeon_RX_6900_Linux_tests.md` (IDs `1INJNrFRFewQGLsDNp5iO5NonntfIbk-f`, `1gHw9RWBLeTo4ZREwhTAAJ24Cnub4QLhL`), `orquestra/maquinas.json` (`130EiljhPERYr7_meXOBVtGnpjRGV2l2Z`, `1gdguuZcoNSs4SJDyd1E6n8N6onb0ZsGP`; the second has an extra `placa_mae` field), `orquestra/README.md` and `rx6900xt_usb/README.md`. Do not delete anything. Report to the owner which copy the local folder exposes, and let the owner remove the extras.

## Ground truth used by these fixes
- Two separate PCIe cables feed the card (owner, 2026-10-15). PSU: Corsair RM1000x.
- Logs now in `temp\gpu_test\`: `memtest_vulkan.log`, `furmark.log`, `llama_bench_vulkan.log`, `test_results.log`, `dmesg_previous_boot.log`, `dmesg_current_boot.log`.
- FurMark never rendered (`cannot open default display!`). memtest_vulkan: ~12.5 min, 12,942 iterations, 0 errors, no size argument exists. llama-bench 0.5B: ~1–2 s of GPU work, passed. llama-bench 3B: pp512 completed, machine reset during pp1024 (seconds of compute). `run_benchmarks.sh` aborted at step 2 in all 3 logged runs. `dmesg_previous_boot.log` is a ~23 s boot (clean `systemd-shutdown`), not the crash session. BAR = 256M in Linux. Kernel changed 6.12.107 → 6.12.111. There is **no** power/temperature-under-load data in any log; the figures 272–284 W, 56/75 °C, 1174 RPM and "3.00 GHz" exist only in chat narrative.
- Steward opinion to record: do **not** flash another vendor's VBIOS.

---

## A. `AMD_Radeon_RX_6900_Linux_tests.md` (dossier)

**D-1 — §0 Resumo.** Replace items 2, 3 and 4 with:
> 2. **Passou:** teste de VRAM (`memtest_vulkan`, ~12,5 min, 12.942 iterações, 0 erros, 448 GB/s escrita / ~430 GB/s checagem) [medido, log]; uma rajada de cálculo com modelo de 0,5B (~1–2 s de GPU, `llama-bench`) [medido, log]. **Sem log:** vkmark. **Não rodou:** FurMark (log: `cannot open default display!`).
> 3. **Temperatura não explica a falha** no log do Windows de 2026-08-17 (hot spot 58 °C no último segundo). No Linux **não existe telemetria sob carga em nenhum log**; os valores 272–284 W / 56 °C / 75 °C citados antes eram relato verbal e foram retirados.
> 4. **A falha no Linux ocorreu após poucos segundos de computação** (`llama-bench` Qwen2.5-3B: pp512 terminou, a máquina reiniciou durante pp1024), **não em carga sustentada**. O teste de VRAM de ~12,5 min não reiniciou. Potência média por si só não está estabelecida como causa nem descartada (não há medição de potência sob carga). Hipótese: pico de corrente no trilho do núcleo em cálculo denso [hipótese].

Replace item 5 with:
> 5. **Ainda não foi medido:** se um limite de potência (50–90 %), um teto de clock ou outro estado de PCIe/ReBAR evita a falha; o tempo até a falha com telemetria transmitida; qualquer teste com outra fonte/placa-mãe. O cabeamento já está resolvido: **dois cabos PCIe separados** (dono, 2026-10-15). Plano de execução: `rx6900xt_usb/TEST_PLAN_LINUX_2026-10-15.md`.

**D-2 — §1 tabela.** Row "Placa": append ` — subsistema 1eae:6901, variante "Black" (lspci)`. Row "VBIOS": replace the value with `113-69XB6SSB1-D01 (ATOM BIOS; lido no Linux e no FurMark)`; keep the SMU text. Row "Resizable BAR": value becomes `BAR=256M também no Linux (dmesg, 2026-10-15); suportado, desligado`. Row "Limite de potência (PPT)": replace `[relatado no teste Linux]` by `[medido: sensors, test_results.log]`. Row "Cabeamento PCIe": replace the whole value by `**Dois cabos PCIe separados** (dono, 2026-10-15)` and drop "ver seção 7". Row "Fonte": keep, but remove the words `não é a causa do reinício` if present.

**D-3 — §2 linha do tempo.** Replace the 2026-10-15 row "Debian 13 headless instalado... reinício reproduzido com llama.cpp Vulkan num modelo de 3B (seção 3.1)" by `Debian 13 headless no SSD USB; sessão de testes (memtest_vulkan, llama-bench 0,5B/3B). Reinício da máquina durante pp1024 do Qwen2.5-3B (seção 3.1). Logs em temp/gpu_test/`. Add a new row: `2026-10-15 | Dono confirma dois cabos PCIe separados (hipótese de cabo em Y refutada) | dono`.

**D-4 — §3.1 inteira** (título "3.1 Linux (Debian 13...)" até antes de "### 3.2"). Replace with:
```
### 3.1 Linux (Debian 13, headless, Mesa RADV 1.4.305, kernel 6.12.107 → 6.12.111, 2026-10-15)

Ambiente: SSD USB externo, GNOME desligado. Logs brutos em `temp/gpu_test/` (extraídos do SSD USB). Os números abaixo vêm desses logs [medido], exceto onde indicado.

| # | Teste | Resultado no log | Observações |
|---|---|---|---|
| L1 | `memtest_vulkan` v0.5.0 (sem argumento de tamanho; dispositivo 1 = RADV Navi 21) | **PASSou.** 12.942 iterações em ~12,5 min, "Standard 5-minute test PASSed", nenhuma linha de erro | 448 GB/s escrita, ~430–436 GB/s checagem; 10,9 GB escritos / 14,5 GB checados por iteração. Uma janela de 30 s (iterações ~3214→3604) escreveu só 263,6 GB/s (sem explicação). **Sem potência/temperatura sob carga no log.** O argumento `12288` do `run_benchmarks.sh` não existe na ferramenta |
| L2 | `vkmark` | **sem log** | a pontuação 13.966 citada antes não tem evidência |
| L3 | FurMark 2.10.2 `--demo furmark-vk --width 1920 --height 1080 --no-score-box --max-time 15` | **não renderizou**: `cannot open default display!`, encerrou em 1 s | FurMark exige display X/Wayland; as afirmações "1.406 quadros / 95 FPS / 98 % de GPU" não têm log |
| L4 | `llama-bench` Qwen2.5-0.5B Q4_K_M `-p 512 -n 128 -r 3` | **passou**: pp512 18.469,9 ± 2.587 t/s, tg128 552,4 ± 36 t/s | tempo total de GPU ≈ 1–2 s (calculado dos t/s); a duração "~10 s" citada antes não tem evidência |
| L5 | `llama-bench` Qwen2.5-3B Q4_K_M `-p 512,1024 -n 128 -r 2` | pp512 terminou (4.266,9 ± 37,3 t/s); **a máquina reiniciou durante pp1024** | poucos segundos de cálculo; não é "sustentado". Sem log de kernel do instante (ver abaixo) |

Registros do sistema: `test_results.log` mostra `run_benchmarks.sh` executado 3× (08:20, 09:15, 12:10), **abortado no passo 2 em todas** (`CMake Error ... memtest_vulkan ... CMakeLists.txt`: a ferramenta é Rust/cargo, e depois `destination path already exists`); os passos 3–5 nunca rodaram, portanto os testes acima foram digitados à mão e os comandos exatos de L1–L3 não estão registrados. `dmesg_previous_boot.log` é um boot de ~23 s (kernel .107) que termina em `systemd-shutdown`: **não contém a sessão do reinício**; o log de kernel do instante da falha não está disponível (verificar `journalctl --list-boots`). `dmesg_current_boot.log` (kernel .111): sem erros amdgpu/AER/MCE; `FAT-fs (sda5): Volume was not properly unmounted` (provável ESP). Os horários dos logs não batem com o `date` dos scripts: usar tempos monotônicos.
```
**D-5 — §4 matriz.** Row "Superaquecimento": base becomes `W2 (hot spot 58 °C na queda). No Linux não há telemetria sob carga`. Row "VRAM defeituosa": situação `sem erros no teste de ~12,5 min`; base `L1 (a ferramenta não permite escolher o tamanho)`; remove "refutada para 12 GB". Row "Potência média alta": situação `não determinada`; base `não há medição de potência sob carga em nenhum log; L1 não reiniciou`. Row "Fonte ou cabeamento": replace with two rows: `Cabeamento | **refutado** (hipótese de cabo em Y) | dono: dois cabos separados` and `Fonte | não isolado | RM1000x; nenhum teste com outra fonte`. Row "Proteção elétrica ... transientes": append to the base `a falha no Linux ocorreu em poucos segundos de pp1024 do 3B`. Replace the sentence "padrão observado em 5 eventos" by "padrão observado em W1–W3 e L5".

**D-6 — §5.3 (verde).** Replace the table with: `Vídeo para o monitor / desktop / 4K-HDR` (uso diário); `Teste de VRAM` (memtest_vulkan ~12,5 min, 12.942 iterações, 0 erros); `Rajada de cálculo` (~1–2 s de GPU com modelo de 0,5B; tempo máximo seguro desconhecido). Delete the rows "3D leve — vkmark" and "FurMark curto" (sem log / não rodou) and list them under §5.2 as `[não testado]`.

**D-7 — §6.** Item 1: replace by `Feito: logs copiados para temp/gpu_test. Falta: extrair o log de kernel do boot do reinício (journalctl --list-boots) e confirmar journal persistente.` Item 3: replace "dois cabos PCIe separados da fonte; " by "" (já são dois). Delete the sentence about "chave de BIOS dupla" or mark `[alegado no relatório de 2026-07-21, não verificado nesta unidade]`.

**D-8 — §7 tabela.** E1: replace the row by `E1 | Cabeamento PCIe | **resolvido: dois cabos separados** (dono, 2026-10-15)`. E2: change the load to `Qwen2.5-3B pp1024 (gatilho conhecido), 50→100 %`. E5: change to `tempo até a falha na configuração padrão com telemetria transmitida e netconsole`. Add the line below the table: `Plano executável: rx6900xt_usb/TEST_PLAN_LINUX_2026-10-15.md.`

**D-9 — §8 scripts.** `run_benchmarks.sh` row: Estado → `**Abortado no passo 2 nas 3 execuções registradas** (memtest_vulkan é Rust/cargo, não CMake); passos 3–5 nunca rodaram; os testes relatados não vieram deste script. Não reutilizar.` (see also F-1.) `setup_rxtest.sh` row: keep the ⚠ text.

**D-10 — §9.** Add a row: `temp/gpu_test/*.log | logs brutos da sessão Linux de 2026-10-15 (memtest_vulkan, furmark, llama_bench_vulkan, test_results, dmesg anterior/atual)`.

**D-11 — §6 final paragraph / VBIOS.** Replace the paragraph that begins "O que este plano **não** propõe:" by:
> O que este plano **não** propõe: undervolt agressivo como solução padrão, mexer em `ppfeaturemask` sem registrar, desligar o TDR para "esconder" a falha, ou **flash de VBIOS de outra marca (Plano D do relatório de 2026-07-21) — não recomendado** pelo mantenedor deste registro: exige forçar a gravação com SSID diferente, pode inutilizar a placa, e nada medido aponta o VBIOS (a falha ocorre em segundos de cálculo, memória e temperatura sem erro); o relatório não cita fonte que o implique nesta unidade.

**D-12 — Histórico (append at the end of "## Histórico deste arquivo").**
> - 2026-10-15 (tarde): corrigido com os logs reais de `temp/gpu_test`: FurMark não rodou; vkmark sem log; memtest ~12,5 min (a ferramenta não tem argumento de tamanho); rajada de 0,5B ≈ 1–2 s; reinício do 3B ocorreu em poucos segundos de pp1024; telemetria sob carga citada antes não consta em nenhum log e foi retirada; `dmesg_previous_boot.log` não é a sessão do reinício. Cabeamento: dois cabos separados (dono).
> - 2026-10-15: **divergência registrada** entre `run_benchmarks.sh` e os testes relatados: o script (Qwen3.5-4B, `memtest_vulkan` via CMake) abortou no passo 2 em 3/3 execuções; os testes foram feitos à mão (Qwen2.5-0.5B/3B em `/opt/gpu_test/models/`, FurMark e vkmark fora do script).
> - 2026-10-15: **opinião do mantenedor:** não gravar VBIOS de outra marca (resumo em §6).

---

## B. Other files

**F-1 — `run_benchmarks.sh` (root).** Insert as line 2 (after the shebang):
`# DEPRECATED 2026-10-15: aborts at step 2 (memtest_vulkan is a Rust/cargo project, not CMake; then "destination path already exists" under set -e). The reported tests were not produced by this script. Use rx6900xt_usb/TEST_PLAN_LINUX_2026-10-15.md.`

**F-2 — `setup_rxtest.sh` (root).** Insert after the shebang: `# WARNING 2026-10-15: appends "PermitRootLogin yes" and "PasswordAuthentication yes" to sshd_config, contradicting the mission (key-only, no root). Do not re-run. Audit with: sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'.`

**F-3 — `orquestra/maquinas.json`, machine `pc`.**
- `fonte`: replace `não é a causa do reinício` by `causa do reinício não estabelecida` (keep the other words, including `single rail 12V 83.3A`).
- `gpu`: replace `memtest_vulkan ~15 min, 0 erros), 3D leve, rajada de cálculo de ~10 s` by `memtest_vulkan ~12,5 min, 0 erros; rajada de ~1–2 s com modelo 0,5B; reinício durante pp1024 do Qwen2.5-3B em poucos segundos de cálculo`. Keep the pointer to the dossier. Preserve valid JSON (validate with a JSON parser after editing).

**F-4 — `rx6900xt_usb/README.md`.** Replace the "Outcome:" sentence by `Outcome (logs em temp/gpu_test): memtest_vulkan ~12,5 min PASS; llama-bench 0,5B (~1–2 s) PASS; llama-bench 3B: reinício durante pp1024 após poucos segundos de cálculo; FurMark não rodou (sem display); vkmark sem log.` Replace "Test logs are on the test Linux, not on Drive: ..." by `Test logs: RTX_Ryzen9/temp/gpu_test/. Missing: the kernel log of the crash boot (see TEST_PLAN A2).` Add under "Scripts": `Plan for the next session: TEST_PLAN_LINUX_2026-10-15.md; doc fixes: PHASE0_DOC_FIXES_2026-10-15.md.`

**F-5 — `temp/updater/RX 6900 XT USB Linux test mission.md`.** Insert at the top: `> STATUS 2026-10-15: OUTDATED. This is the 2026-10-13 preparation report. Linux was installed afterwards and a first GPU session ran; see AMD_Radeon_RX_6900_Linux_tests.md.` Mark the sentence "Linux is not installed yet, and no GPU test has run." as `[superseded]` and the last paragraph ("From there I SSH in and do ... the power sweep") as `[superseded by TEST_PLAN_LINUX_2026-10-15.md]`.

**F-6 — `temp/updater/RX6900XT_USB_TEST_AGENT_MISSION.md`.** Insert at the top: `> STATUS 2026-10-15: original safety brief, still binding. Active procedure: rx6900xt_usb/TEST_PLAN_LINUX_2026-10-15.md. The referenced file CONTEXTO_IT_SERVIDORES_E_TESTE_RADEON.md was not found on Drive; use AMD_Radeon_RX_6900_Linux_tests.md.` Replace the sentence "The GPU has a documented history of abrupt restarts under sustained load. The existing record attributes this to the GPU and says the power supply was investigated." by `The GPU has a documented history of abrupt whole-machine restarts under GPU load (Windows: MSFS, DirectML; Linux 2026-10-15: llama.cpp Vulkan, Qwen2.5-3B, within seconds of compute). The cause is not established; see AMD_Radeon_RX_6900_Linux_tests.md §4.` Do not change anything else (another agent edited this file today: BIOS 3636).

**F-7 — `orquestra/revisoes/2026-10-01/pesquisa/auditoria_codex.md`.** Replace the whole paragraph under "### p15: reinício da RX 6900 XT" by:
> Em 2026-10-01 o dono registrou "defeito de fábrica, fora da garantia, assunto encerrado". **Em 2026-10-15 o dono reabriu a investigação**: a causa não está estabelecida (o "defeito de fábrica" é hipótese, não medição) e a fonte Corsair RM1000x não foi isolada como causa nem descartada. Estado, testes e plano: `AMD_Radeon_RX_6900_Linux_tests.md`. Registro anterior: memória `amd-gpu-reboots-under-sustained-load` (projeto SJC, 2026-09-15: um lote de 70 imagens em DirectML derrubou a máquina; 3 imagens passaram).

**F-8 — `orquestra/revisoes/2026-10-01/pesquisa/p15.md` (Codex).** Insert as the first line: `> NOTA 2026-10-15: a seção sobre a RX 6900 XT é pesquisa de 2026-10-01. Fatos novos: dois cabos PCIe separados já em uso (dono); logs em temp/gpu_test; o reinício no Linux ocorreu em poucos segundos de pp1024 do Qwen2.5-3B; "usável com llama.cpp Vulkan" NÃO está demonstrado nesta placa. Ver AMD_Radeon_RX_6900_Linux_tests.md.` In the bullet that begins "For the RX 6900 XT reboot, first run stock clocks/voltage, reseat the card, use two separate PCIe power leads" delete "use two separate PCIe power leads, " (already the case).

**F-9 — Gemini `p15.md` (folder `.../pesquisa/` under the Antigravity subfolder, fileId `1rsmQmlrKY7YZG12Rs4a7Nk2vR54FiSJn`).** Insert as the first line: `> NOTA 2026-10-15 — NÃO USAR a parte da RX 6900 XT: a causa ("transientes derrubando a OCP da fonte") e a correção ("undervolt / -10 % de limite") são afirmações sem fonte nem medição; undervolt está excluído pela missão; dois cabos separados já estão em uso. Fonte da verdade: AMD_Radeon_RX_6900_Linux_tests.md.` Do not delete the original text (historical record).

**F-10 — `G14_Headless_Server_Config/DOSSIE_GRID_G14.md`.** Replace `RX 6900 XT **com defeito** (reinicia sob carga): **nunca** usada` by `RX 6900 XT (reinicia sob cálculo pesado na GPU; causa não estabelecida; ver AMD_Radeon_RX_6900_Linux_tests.md): não usada para cálculo em produção`.

**F-11 — `00_README_PROJETO.md`.** The scope sentence "Este projeto NÃO cobre o desktop de jogos (Ryzen 9 5900X / RX 6900 XT)." makes agents refuse the card's work. Replace by: `Este projeto NÃO cobre o desktop de jogos (Ryzen 9 5900X), exceto os registros e testes da RX 6900 XT, mantidos em AMD_Radeon_RX_6900_Linux_tests.md e rx6900xt_usb/.`

**Left unchanged on purpose** (historical or already correct): `CODEX_pesquisa.md` and `ANTIGRAVITY_pesquisa.md` (task prompts as sent on 2026-10-01), `suplemento_atualizador_desempenho.md`, `network_topology.md`, `PROMPT_NOVO_CHAT.md`, `ENVIRONMENT_REPORT.md`, `Claude Cloud usage costs.md`.
Outside the project folder, not editable here: the 2026-07-21 research report (`AMD Radeon 6900 XT/Relatório Pesquisa erro driver.docx` and the Google Doc "MSFS 2020 Travando com RX 6900 XT"). Its causal claims are unsourced; its Plan D is not recommended (D-11). Tell the owner so it is not used as a procedure.
