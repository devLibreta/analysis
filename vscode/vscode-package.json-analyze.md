# VS Code `package.json` 분석 보고서

분석 대상은 `vscode/package.json` (`name: code-oss-dev`, `version: 1.143.0`)입니다. 일반 라이브러리 매니페스트가 아니라 VS Code 본체(데스크톱·웹·CLI·내장 확장)를 빌드하고 검증하기 위한 루트 정의서로서 어떤 구조를 가지는지 분석합니다.

| 분석 항목 | 결과 요약 |
|---|---|
| **정체** | 배포 패키지가 아닌 OSS 개발용 루트 매니페스트(`private: true`) |
| **규모** | `scripts` 88개, `dependencies` 70개, `devDependencies` 102개 |
| **빌드 축** | `gulp` + `build/next` 이원화, `compile`과 `watch` 계열 분리 |
| **의존성 축** | 제품 런타임은 `@vscode/*` 네이티브·`xterm`·터널·AI SDK 중심 |
| **통제 축** | `overrides`로 전이 의존성 고정, `allowScripts`로 설치 스크립트 허용 통제 |

## 1. 분석 범위 및 방법

- 원본 `vscode/package.json` 전체(330행)를 대상으로 필드별 역할과 상호 관계를 분석합니다.
- `gulpfile.mjs`, `build/`, `product.json`의 실제 구현은 범위에 포함하지 않으며, 매니페스트에 드러난 설계 의도만 다룹니다.
- 스크립트 개수와 의존성 개수는 `node`로 `package.json`을 직접 파싱하여 확인한 값입니다.

## 2. 기본 식별자 분석

- **`name: code-oss-dev`는 OSS 개발 빌드용 명칭입니다.** 스토어 배포명과 분리되어 있으며, npm 발행이 전제되지 않은 이름입니다.
- **`private: true`와 `license: MIT`가 병존합니다.** 라이선스는 MIT이나 배포 경로가 npm이 아니므로 `private`로 오발행을 차단한 구조입니다.
- **`type: module`과 `main: ./out/main.js`는 ESM 기반 Electron 앱 구조를 나타냅니다.** 진입점이 소스(`src/`)가 아니라 컴파일 산출물(`out/`)이므로, 매니페스트만으로는 실행이 불가능하고 선행 빌드가 필수입니다.
- **`version: 1.143.0`과 `distro` 해시는 제품 버전 식별 체계입니다.** `distro`(`64d8a64e…`)는 `update-distro` 스크립트로 갱신되는 별도 빌드 식별자이며, 일반 semver 운용과 다릅니다.
- **`copilotRuntimeVersion: 1.0.99-1`은 비표준 커스텀 필드입니다.** npm 스펙이 아니라 VS Code-Copilot 간 런타임 호환성을 고정하기 위한 제품 전용 메타데이터입니다.

## 3. `scripts` 구조 분석

88개 스크립트는 역할별로 명확히 분리되어 있습니다.

| 분류 | 대표 스크립트 | 분석 내용 |
|---|---|---|
| **설치 수명 주기** | `preinstall`, `postinstall`, `install-fast` | `build/npm/*.ts`로 위임되며 환경 점검과 고속 설치를 담당합니다. 자동 실행 전제이므로 생략 시 후속 빌드가 성립하지 않습니다. |
| **1회성 빌드** | `compile`, `compile-client`, `compile-copilot`, `compile-web`, `compile-cli` | `compile`은 `npm-run-all2 -lp`로 클라이언트와 Copilot 빌드를 병렬 수행합니다. 타깃별(`web`, `cli`) 파이프라인이 별도로 존재합니다. |
| **감시 빌드** | `watch`, `watch-client`, `watch-extensions`, `watch-copilot`, `watch-web` | 일상 개발용이며 20개 이상이 `watch` 관련입니다. `*d` 접미사(`watchd`, `watch-clientd`)는 `deemon` 기반 백그라운드 실행용이며 `kill-*`, `restart-*`와 쌍을 이룹니다. |
| **빌드 엔진 이원화** | `gulp compile` 계열, `build/next/*.ts` 계열 | 정식 빌드는 `gulp`(`gulpfile.mjs`), 고속 트랜스파일은 `build/next/index.ts`를 사용합니다. `transpile-client`와 `compile-client`는 속도와 엄밀성이 다른 별개 경로입니다. |
| **테스트·검사** | `test-node`, `test-browser`, `smoketest`, `eslint`, `hygiene`, `valid-layers-check` | `npm test` 자체는 실행용이 아니라 안내용 스텁입니다. 단위(`mocha`), 브라우저(`playwright`), E2E(`smoke`), 정적 분석(`eslint`, `tsec`, `layers`)이 영역별로 분리되어 있습니다. |
| **생성·동기화** | `update-grammars`, `download-builtin-extensions`, `export-policy-data`, `codex:gen-protocol` | 소스 직접 수정이 아니라 스크립트 실행으로 산출물을 갱신하는 구조입니다. 프로토콜·문법·정책 데이터의 정합성을 코드가 아닌 생성 절차로 보장합니다. |
| **위임 패턴** | `compile-copilot`, `watch-copilot`, `copilot:setup` | `npm --prefix extensions/copilot run` 형태로 서브 패키지에 위임합니다. 루트는 오케스트레이션이고 실제 구현은 하위 패키지에 존재합니다. |

특이 사항으로 `web` 스크립트는 빌드를 수행하지 않고 `echo`로 `./scripts/code-server`, `./scripts/code-web` 사용을 안내합니다. 매니페스트 내에 퇴역 경로를 안내 문구로 남겨 둔 사례입니다.

## 4. `dependencies` 분석

70개 제품 의존성은 에디터 런타임에 직접 필요한 구성으로 한정됩니다.

- **`@vscode/*` 네이티브·포크군이 중핵입니다.** `ripgrep-universal`, `sqlite3`, `spdlog`, `sudo-prompt`, `native-watchdog`, `windows-registry`, `proxy-agent` 등 OS·Electron 연동 모듈이 다수를 차지합니다. 표준 npm 패키지로 대체가 어려운 제품 고유 계층입니다.
- **터미널 스택은 `@xterm/*` 베타 계열로 고정되어 있습니다.** `xterm` 본체와 `addon-*` 8종이 모두 `beta` 버전대이며, 일괄 버전 운용이 전제된 구조입니다.
- **원격·터널 계열은 `@microsoft/dev-tunnels-*` 5종 세트입니다.** `connections`, `contracts`, `management`, `ssh`, `ssh-tcp`가 함께 고정되어 원격 개발 경로를 구성합니다.
- **AI·에이전트 계열이 제품 의존성에 포함되어 있습니다.** `@anthropic-ai/sdk`, `@github/copilot-sdk`, `@vscode/copilot-api`, `foundry-local-sdk`가 대표적이며, AI 기능이 실험적 확장이 아니라 본체 의존성으로 편입된 상태를 보여줍니다.
- **에디터 코어용으로 `vscode-textmate`, `vscode-oniguruma`, `vscode-regexpp`, `jschardet`, `katex`가 존재합니다.** 구문 강조·정규식·인코딩 감지·수식 렌더링 등 렌더링 파이프라인의 기반 라이브러리입니다.

## 5. `devDependencies` 분석

102개 개발 의존성은 제품에 동봉되지 않는 빌드·검사 인프라입니다.

- **`gulp`와 `gulp-*` 플러그인군이 빌드 본체입니다.** `gulp-azure-storage`, `gulp-sourcemaps`, `gulp-rename` 등 10종 이상이 `gulpfile.mjs` 파이프라인을 구성합니다.
- **타입스크립트 계열이 별칭 설치로 고정되어 있습니다.** `typescript: npm:@typescript/typescript6@^6.0.2`, `@typescript/native: npm:typescript@^7.0.2` 형태이며, `update-build-ts-version` 스크립트가 갱신 경로로 지정되어 있습니다. 임의 버전 상승이 아닌 절차적 갱신이 전제됩니다.
- **검사 도구가 영역별로 분리되어 있습니다.** `eslint`·`typescript-eslint`, `stylelint`(`build/stylelint.ts` 경유), `tsec`, `layersChecker`, `propertyInitOrderChecker`가 각기 다른 결함 축(스타일·보안·아키텍처·초기화 순서)을 담당합니다.
- **테스트 인프라는 `mocha`·`sinon`·`playwright`·`c8`/`istanbul-*` 조합입니다.** Node 단위 테스트와 브라우저 테스트, 커버리지 측정이 분리된 구성을 보여줍니다.

## 6. 버전 통제와 설치 정책 분석

- **`overrides`는 전이 의존성 강제 고정 장치입니다.** `@typescript-eslint/types: 8.66.0`, `node-gyp-build: 4.8.1`, `koffi: 3.2.1`, `serialize-javascript: ^7.0.3`, `yauzl: ^3.3.1`이 지정되어 있으며, 직접 의존성이 아니라 하위 트리의 버전 드리프트를 상쇄하는 목적입니다. `kerberos@2.1.1` 하위의 `node-addon-api: 7.1.0`처럼 패키지 한정 지정도 포함됩니다.
- **`allowScripts`는 설치 스크립트 실행 허용 목록입니다.** `native-keymap`, `node-pty`, `kerberos`, `@vscode/sqlite3`, `@vscode/spdlog` 등 네이티브 빌드가 필요한 패키지는 `true`, `ssh2`, `koffi`, `es5-ext`, `husky` 등은 `false`로 차단되어 있습니다. 보안 표면을 줄이면서 필수 네이티브 모듈만 허용한 정책입니다.
- **`optionalDependencies`는 사실상 Windows 전용 1건입니다.** `windows-foreground-love: 0.6.1`만 포함되어 있으며, 부재 시에도 실행이 성립하는 선택적 구성입니다.

## 7. 관찰 사항 및 리스크

- **빌드 진입점이 이중화되어 학습·유지 비용이 높습니다.** `gulp` 기반 정식 경로와 `build/next` 기반 고속 경로가 공존하므로, 문제 발생 시 어느 경로의 산출물인지 먼저 식별해야 합니다.
- **`watch` 계열의 비대화가 관찰됩니다.** 88개 중 20개 이상이 감시·백그라운드 변형이며, `deemon` 프로세스 잔류 시 포트·파일 감시가 충돌할 가능성이 있습니다.
- **네이티브 모듈 비중이 높아 설치 환경 의존성이 큽니다.** `@vscode/*` 네이티브군은 OS·아키텍처별 바이너리를 요구하므로 `allowScripts` 차단이나 누락된 빌드 도구에서 설치 실패가 발생하기 쉽습니다.
- **AI SDK가 제품 의존성에 편입되어 공급망 관리 범위가 확대되었습니다.** `@anthropic-ai/sdk`, `@github/copilot-sdk`, `foundry-local-sdk`의 버전·보안 공지가 에디터 본체의 릴리스 리스크에 직접 연결됩니다.
- **`@xterm/*` 베타 고정과 `playwright-core` 알파 지정(`1.61.0-alpha-2026-06-04`)은 불안정 채널 의존을 의미합니다.** 안정 릴리스와 동일한 기준으로 자동 업데이트를 적용하기 어렵습니다.

## 8. 결론

이 `package.json`은 패키지 배포 명세가 아니라 대형 데스크톱 애플리케이션의 빌드 오케스트레이션 정의서입니다. 제품 식별, 88개 스크립트로 구성된 빌드·감시·검사 체계, 네이티브 중심의 런타임 의존성, `overrides`와 `allowScripts`로 구성된 공급망 통제라는 네 축으로 이루어져 있습니다. 후속 분석은 `gulpfile.mjs`와 `build/` 하위, `extensions/copilot/package.json`, `product.json`과의 역할 분담으로 확장하는 것이 적절합니다.

## 참고 자료

- 원본: `vscode/package.json` (분석 기준 330행)
- 빌드 정의: `vscode/gulpfile.mjs`, `vscode/build/`
- 기여 절차: `vscode/CONTRIBUTING.md`
- 에이전트 작업 규칙: `vscode/AGENTS.md`, `vscode/.github/copilot-instructions.md`
- 제품 브랜딩 설정: `vscode/product.json` (`package.json`과 분리 관리됨)
