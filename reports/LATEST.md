# LATEST — облако-исполнитель

## T3-cert-2 — BLOCKED (нужна 1 команда от локального Claude с админ-токеном)
Обновлено: 2026-09-26 ~11:45 UTC

### Итог в одну строку
Сертификат не выпускается потому, что **DNS-проверка на стороне GitHub не проходит**, хотя тот же код
проверки снаружи проходит. Гипотезы «нет www» и «нет верификации домена» — **не причина** (по докам GitHub).
Главная гипотеза: инфраструктура GitHub Pages не достукивается до NS reg.ru (российские адреса) → `InvalidDNSError`.
Подтверждается одной командой (ниже), делать её должен локальный Claude — у облака нет админ-прав на Pages API.

### Что проверено (факты)
- `GET /repos/Pazaks/pazak-update/pages` (с Actions, токен pages:read), 24.09 и 26.09:
  `cname=pazak.ru, status=built, https_certificate=null, https_enforced=false, protected_domain_state=null`.
  `https_certificate=null` = GitHub даже не поставил задачу на выпуск сертификата.
- crt.sh: для pazak.ru нет ни одного сертификата, выпущенного после переезда на GitHub (последний — 23.08, со старого сервера).
  Неудачные попытки Let's Encrypt в CT не попадают, т.е. процесс оборвался ДО Let's Encrypt — на DNS-проверке GitHub.
- Гем `github-pages-health-check` 1.19.2 (тот же код, что у GitHub) на раннере Actions, режимы nameservers
  default / authoritative / public: `check! = OK`, `valid?=true`, `https_eligible?=true`, `caa_error=null`,
  `dns_resolves?=true`, `served_by_pages?=true`. Т.е. снаружи домен полностью годен для сертификата.
- NS reg.ru (ns1/ns2) напрямую, A/AAAA/CNAME/MX/CAA/TXT/NS/SOA × UDP/TCP: все NOERROR за 0.2–0.4 с (1.0 с максимум).
- Сейчас pazak.ru по HTTPS отдаёт серт `*.github.io` → старые приложения падают на ошибке сертификата.
  По HTTP (без S) страница отдаётся (206).

### Гипотезы из задачи
1. **Нет www → блокирует серт apex: НЕ подтверждено.** Док «Securing your site with HTTPS» / «Verifying the DNS
   configuration»: мешать могут только ЛИШНИЕ записи (доп. A/AAAA/ALIAS у `@`, CNAME на www, указывающий не туда).
   www для apex — «recommended», не required. Отсутствие www не упомянуто как блокер.
   Добавить `www CNAME pazaks.github.io` в reg.ru безвредно и рекомендуется GitHub, но проблему, скорее всего, не решит.
2. **Верификация домена (protected_domain_state=null): НЕ нужна.** Док «Verifying your custom domain»: верификация
   только защищает от захвата домена чужим репо; на выпуск сертификата не влияет.

### Решающая проверка (локальному Claude, с токеном Pazak)
```
gh api repos/Pazaks/pazak-update/pages/health        # первый вызов может вернуть 202 — повторить через ~30 с
```
Смотреть `domain.reason` / `domain.dns_resolves?` / `domain.https_eligible?`.
- Если `reason` = `InvalidDNSError` (или dns_resolves=false), а снаружи всё OK (см. выше) →
  **подтверждено: GitHub со своей стороны не может получить DNS pazak.ru.** Лечение — ниже.
- Если `reason` пустой и `https_eligible?=true`, а `https_certificate` всё равно null → застряла очередь на стороне
  GitHub → только GitHub Support (черновик ниже).

### Лечение, если подтвердится (решение Pazak)
Перенести DNS-хостинг pazak.ru с NS reg.ru к DNS-провайдеру вне РФ (регистратор остаётся reg.ru, меняются только NS):
Cloudflare (только DNS, «серое облако», без проксирования) / deSEC / Hurricane Electric DNS.
⚠ Перед сменой NS перенести ВСЕ записи зоны 1:1 (в зоне есть и поддомены, не только `@` — список в STATE),
иначе они отвалятся. После смены NS (распространение до ~24 ч) — одна перепривязка домена в Pages.

### Черновик в GitHub Support (если проверка покажет пустой reason)
> Subject: GitHub Pages: HTTPS certificate never provisioned for custom apex domain (https_certificate is null)
>
> Repository: Pazaks/pazak-update (legacy build from `main`), custom domain `pazak.ru` (apex).
> DNS: 4 A records to 185.199.108–111.153, no AAAA, no CAA, no DNSSEC. Authoritative NS respond over UDP/TCP in <0.5 s.
> `github-pages-health-check` 1.19.2 run from GitHub Actions returns valid/https_eligible for all nameserver modes.
> Still, `GET /repos/Pazaks/pazak-update/pages` returns `"https_certificate": null` since 2026-09-24, and the Pages
> health endpoint earlier reported "Domain's DNS record could not be retrieved (InvalidDNSError)".
> The domain was removed/re-added many times (last: 2026-09-26 ~11:38 UTC) without effect.
> Could you check why certificate provisioning is not queued for this domain, and whether your DNS check can
> reach the authoritative nameservers ns1.reg.ru / ns2.reg.ru?

### Что облако сделало в main (для журнала)
- 26.09 ~11:37 UTC `Delete CNAME` → сборка Pages OK → ~11:39 UTC `Create CNAME` (pazak.ru). Файл CNAME побайтно
  как был (перепривязка по разрешению Pazak до получения T3-cert-2). Больше в main не пушу.

### Риск на потом (не блокер задачи)
Подсеть GitHub Pages `185.199.108.0/22` есть в списках antifilter (allyouneed/ipsum — IP из реестра РКН).
Даже с сертификатом часть провайдеров РФ может не пускать на pazak.ru@GitHub Pages. Проверить с реального
устройства в РФ **без VPN**, когда серт появится.
