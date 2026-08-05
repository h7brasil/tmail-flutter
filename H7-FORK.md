# H7 Mail — fork de rebranding do Twake Mail

Fork de [linagora/tmail-flutter](https://github.com/linagora/tmail-flutter) (AGPL-3.0),
rebrandeado como **H7 Mail** e apontado para o servidor JMAP do Stalwart em
`mail.h7brasil.com`.

Base: `v0.33.0`. Branch de trabalho: `h7-rebrand`.

## Licença e marca

O código é AGPL-3.0 e continua AGPL-3.0. A licença **não cede a marca "Twake"** —
por isso o app foi renomeado e os identificadores trocados. Este fork é público,
o que já satisfaz a obrigação da AGPL §13 de disponibilizar o código correspondente
a quem receber o binário.

## O que mudou

| Item | Antes | Depois |
|---|---|---|
| Nome (Android) | Twake Mail | H7 Mail |
| Nome (iOS) | Twake Mail | H7 Mail |
| `applicationId` | `com.linagora.android.teammail` | `com.h7brasil.mail` |
| Bundle iOS | `com.linagora.ios.teammail` | `com.h7brasil.ios.mail` |
| Ícones Android | Twake | marca H7 (5 densidades + adaptive) |
| Splash / wordmark | Twake | marca H7 (`splash`, `branding`, variantes Android 12) |
| Ícone de notificação | Twake | silhueta H7 |
| Logos dentro do app | `ic_logo_twake_welcome.svg` e cia. | marca H7 |
| Tela de boas-vindas | Twake SaaS | pulada (vai direto ao login) |
| `env.file` | localhost / FCM on | `mail.h7brasil.com` / FCM off |

### A tela de boas-vindas foi pulada de propósito

`TwakeWelcomeController.onInit` chama `handleUseCompanyServer()` num
`addPostFrameCallback`. A tela original oferece **"Create Twake ID"**
(`sign-up.twake.app`) e **"Sign in"** (`jmap.twake.app`) — os dois apontam para o
SaaS da LINAGORA e não servem para quem usa servidor próprio. O app abre
direto no formulário que resolve o SRV `_jmap._tcp`.

Trade-off conhecido: o botão "voltar" a partir do login não sai mais do app,
porque volta para a welcome, que redireciona de novo. Sair pelo botão home.

### Rebranding visual é maior do que parece

Trocar o ícone do launcher **não basta**. Estas são superfícies independentes,
e cada uma manteve a marca da Twake até ser trocada uma a uma:

1. `mipmap-*/ic_launcher*` — ícone do launcher
2. `drawable/ic_launcher_background` + `drawable-*/ic_launcher_foreground` — adaptive icon
3. `drawable-*/splash` e `android12splash` — splash nativo
4. `drawable-*/branding` e `android12branding` — wordmark do splash
5. `drawable-*/notification_icon` e `drawable/ic_large_notification`
6. `assets/images/ic_logo_*.svg` — logos renderizados pelo Flutter dentro do app

O item 6 é o que aparece na tela de boas-vindas e **não vem dos recursos do
Android** — é asset do Flutter, invisível para qualquer inspeção de `res/`.

Nota de verificação: o AAPT2 recomprime PNG no empacotamento, então comparar
assets do APK por hash dá falso negativo. Compare por dimensão ou visualmente.

### O `namespace` do Android continua `com.linagora.android.tmail` — de propósito

Os `MethodChannel` usam essa string como literal **nos dois lados**
(`android_selection_handles_manager.dart` e `AndroidSelectionHandles.kt`). O
`namespace` só define o pacote da classe `R`; quem identifica o app no aparelho e
na loja é o `applicationId`. Trocar o namespace exigiria mover os pacotes Kotlin
sem nenhum ganho visível.

### Assinatura do release

`android/app/build.gradle` cai na keystore de debug quando `key.properties` não
existe — sem isso, `flutter build apk --release` falharia em qualquer máquina que
não tenha a keystore da LINAGORA. Com os secrets configurados, o CI escreve o
`key.properties` e o APK sai assinado de produção.

**Chave de produção (gerada em 2026-08-04):**

| | |
|---|---|
| Arquivo | `~/.h7mail-signing/h7mail-release.p12` (fora do repositório) |
| Formato | PKCS12 — gerado com `openssl`, já que não havia JDK na máquina |
| Chave | RSA 4096, SHA-256 |
| Alias | `h7mail` |
| Validade | 2026-08-04 → 2053-12-20 (o Google Play exige até pelo menos 2033) |
| SHA-256 | `76:0E:B4:5B:17:B0:12:C9:84:BB:80:41:C4:95:85:AC:D2:63:F8:63:45:A4:56:5D:DD:EE:0E:BD:71:75:6D:0E` |

Secrets do repositório: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`,
`ANDROID_KEY_ALIAS`.

**`storeFile` precisa ser caminho absoluto no CI.** O `key.properties` é lido de
`android/` (`rootProject`), mas o `file()` que resolve o `storeFile` executa no
projeto `android/app/`. Caminho relativo aponta para o lugar errado e o build
morre em `validateSigningRelease`.

### Backup da keystore — leia antes que seja tarde

Perder esta chave significa **nunca mais conseguir atualizar o app** para quem já
o instalou: o Android recusa uma atualização assinada por chave diferente. Não há
recuperação — nem pelo Google, nem por nós. A única saída seria publicar outro app
e pedir a todos que desinstalem e reinstalem.

O que fazer, hoje:

1. Guarde `h7mail-release.p12` **e** a senha (`~/.h7mail-signing/.password`) num
   gerenciador de senhas ou cofre. Os dois juntos, em lugar diferente da máquina.
2. Não confie nos secrets do GitHub como backup — o GitHub não permite ler um
   secret de volta, só sobrescrever.
3. Se um dia usar o Google Play, ative o **Play App Signing**: o Google passa a
   guardar a chave de assinatura final e a sua vira apenas chave de upload,
   que é substituível caso se perca.

Para restaurar noutra máquina, ou reenviar aos secrets:

```bash
base64 -i h7mail-release.p12 | gh secret set ANDROID_KEYSTORE_BASE64 --repo h7brasil/tmail-flutter
gh secret set ANDROID_KEYSTORE_PASSWORD --repo h7brasil/tmail-flutter < .password
printf 'h7mail' | gh secret set ANDROID_KEY_ALIAS --repo h7brasil/tmail-flutter
```

### Quem tem escrita no repositório consegue extrair a chave

Secrets são mascarados no log e não são expostos a PRs de forks, mas qualquer um
com permissão de escrita pode abrir um workflow que exfiltre o valor. Hoje isso
significa só você. Ao adicionar colaboradores, considere mover a assinatura para
um ambiente protegido com revisão obrigatória, ou tirá-la do CI.

## Push notifications não funcionam — e não é bug de configuração

O Stalwart não implementa nenhum dos dois mecanismos que o Twake usa:

- **FCM** exige a extensão Firebase do Apache James.
- **WebSocket push** é gated em `com:linagora:params:jmap:ws:ticket`
  (`web_socket_datasource_impl.dart`), extensão da LINAGORA.

O Stalwart oferece o `urn:ietf:params:jmap:webpush-vapid` padrão, mas o tmail
não implementa VAPID. Por isso `FCM_AVAILABLE` e `IOS_FCM` estão vazios: o app
sincroniza ao abrir, sem notificação em segundo plano. Fechar essa lacuna seria
contribuir suporte a VAPID no upstream.

## Como o app acha o servidor

Não é pelo `SERVER_URL` — no mobile ele é ignorado:

```dart
// login_controller.dart
Uri? get _currentBaseUrl => PlatformInfo.isWeb
    ? Uri.tryParse(AppConfig.baseUrl)
    : _baseUri;
```

O mobile resolve o SRV `_jmap._tcp.<domínio do e-mail>` (RFC 8620). Para
`h7brasil.com` o registro já existe, publicado pela integração Cloudflare do
Stalwart:

```
_jmap._tcp.h7brasil.com.  SRV  0 1 443 mail.h7brasil.com.
```

Como o Stalwart não serve `/.well-known/webfinger`, o app cai no formulário de
senha (basic auth), que o Stalwart aceita. **Use um App Password do Stalwart**,
não a senha principal da conta.

## Build

Pelo GitHub Actions (nada a instalar):

```
Actions → "H7 Mail — Android APK" → Run workflow
```

O APK sai como artifact. `--split-per-abi` gera um APK por arquitetura; para
celular moderno use o `app-arm64-v8a-*.apk`.

Local, se um dia quiser:

```bash
./scripts/prebuild.sh
flutter build apk --release --split-per-abi
```

Requer Flutter 3.38.9 (o upstream fixa essa versão nos workflows), JDK 17 e o
Android SDK.

## Pendências do iOS

Nada disso bloqueia o Android.

1. `DEVELOPMENT_TEAM` no `project.pbxproj` continua `KUT463DS29` (LINAGORA) —
   trocar pelo Team ID da H7.
2. `AppConfig.iOSKeychainSharingGroupId` está com o placeholder
   `REPLACE_WITH_APPLE_TEAM_ID`. Precisa virar `<TEAM_ID>.com.h7brasil.ios.mail.shared`
   e bater com o entitlement do target `Runner`.
3. O target `TwakeMailNSE` (Notification Service Extension) pode ser removido —
   só serve para push, que não funciona aqui.
4. Ícones e splash do iOS ainda são os da Twake (`ios/Runner/Assets.xcassets`).
5. Conta Apple Developer: US$ 99/ano. Sem ela, build expira em 7 dias.

## Manutenção

Para trazer atualizações do upstream:

```bash
git fetch upstream
git rebase upstream/master
```

Espere conflitos em `project.pbxproj`, `build.gradle` e nos assets de ícone —
são exatamente os arquivos que o rebranding toca.
