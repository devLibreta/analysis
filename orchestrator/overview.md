# REST/SOAP 조합형 오케스트레이터 개요

이 문서는 REST/SOAP를 순차 호출하고 응답을 조합해서 돌려주는 기능을 여러 개 운영하기 위한 `host + plugin` 분리 구조를 정의합니다. 실행 시스템과 로직을 분리해서 플러그인 단위로 추가/삭제하는 것이 목표입니다. 상세 분석은 `analysis/vscode/`에 있습니다.

## Context

전제는 기능마다 외부 REST/SOAP API를 순서대로 호출하고 결과를 조합해서 HTTP로 반환하는 것입니다. 화면이 없으므로 Workbench 렌더러와 확장 호스트 전체를 구동하지 마세요.

VS Code에서 가져올 것은 매니페스트, 생명주기, 기여점 개념뿐입니다. 프로세스 분리 통신과 렌더러 의존은 버리세요.

## Decision

**Host는 스캔과 등록만 담당하므로 로직을 포함하지 마세요.**

- `plugins/*/`를 스캔하고 `plugin.json`을 검증한 뒤 `handler.ts`를 `import()`합니다.
- `rest/soap` 클라이언트, `log/config`를 `ctx`로 주입합니다. 플러그인은 `ctx` 외부에 접근하지 않습니다.
- 라우트 등록과 해제를 Host가 소유합니다. 플러그인은 자신의 `route`를 선언만 합니다.

**Plugin 계약은 두 파일로 고정하세요.**

```text
plugins/<feature>/
  plugin.json
  handler.ts
```

`plugin.json` 예시:

```json
{
  "name": "feature-a",
  "version": "1.0.0",
  "route": "POST /features/a"
}
```

`handler.ts` 예시:

```ts
export async function handle(input: unknown, ctx: HostContext): Promise<unknown> {
  const a = await ctx.rest.get('https://api-a.example.com/items', input);
  const b = await ctx.soap.call('https://soap.example.com/service', a);
  return combine(a, b);
}
```

생명주기는 `scan → validate → activate → register → serve → deactivate → unload` 순서로 진행하세요. 삭제는 폴더 삭제 후 라우트 해제로 처리합니다. 재시작 없는 반영이 필요할 때만 파일 감시와 `import` 캐시 무효화를 추가하세요.

## VS Code에서 볼 소스

| 파일 | 볼 위치 | 용도 |
|---|---|---|
| `src/vs/platform/extensions/common/extensions.ts:344` | `IRelaxedExtensionManifest` (`name`, `version`, `main`, `activationEvents`, `contributes`) | `plugin.json` 설계 참고용으로 보세요. `engines`, `categories`는 무시하세요. |
| `src/vs/workbench/api/common/extHostExtensionActivator.ts:18` | `IExtensionModule` (`activate`/`deactivate`) | 플러그인 생명주기 계약 참고용으로 보세요. `vscode.ExtensionContext` 전체는 가져오지 마세요. |
| `src/vs/workbench/services/extensions/common/extensionsRegistry.ts:61` | `IExtensionPointUser`, `IExtensionPointHandler` | 라우트 등록을 기여점으로 다룰 때 참고용으로 보세요. JSON 스키마 레지스트리는 만들지 마세요. |
| `src/vs/workbench/api/common/extensionHostMain.ts:161` | `ExtensionHostMain` 생성자 (`protocol`, `initData`, `RPCProtocol`) | 복사하지 말아야 할 사례로 보세요. 렌더러 없이는 동작하지 않습니다. |
| `src/vs/workbench/api/node/extensionHostProcess.ts:332` | `connectToRenderer()` | 렌더러 핸드셰이크가 필수임을 확인용으로 보세요. headless 플러그인에는 부적합합니다. |
| `src/vs/workbench/services/extensions/common/rpcProtocol.ts:1` | `RPCProtocol` | 프로세스 분리(B안)를 선택할 때만 보세요. in-process(A안)에서는 불필요합니다. |
| `src/vs/server/node/server.main.ts:1` | 서버 부팅과 확장 디렉터리 준비 | 화면 없는 서버가 확장을 배치하는 방식 참고용으로 보세요. 통째로 재사용하지 마세요. |

## Alternatives

| 안 | 구조 | 선택 기준 |
|---|---|---|
| **A. in-process `import()`** | Host와 플러그인이 한 프로세스 | 기본값으로 사용하세요. 단순하고 이 전제에 충분합니다. |
| **B. `worker_threads`** | 플러그인별 격리 실행 | 플러그인 폭주나 크래시 격리가 필요할 때만 사용하세요. |
| **C. 컨테이너 분리** | 기능별 독립 배포 | 배포 주기가 다를 때만 사용하세요. 게이트웨이가 별도로 필요합니다. |

## Consequences

A안을 사용하면 추가는 폴더 복사, 삭제는 폴더 삭제와 라우트 해제로 끝납니다. 대가로 플러그인 오류가 Host 프로세스에 영향을 주므로 `handle` 단위로 타임아웃과 `try/catch`를 적용하세요.

## Gotchas

- **`handle`에 HTTP 코드를 넣지 마세요.** 상태 코드는 Host가 매핑하고, 플러그인은 데이터와 에러 종류만 반환하세요.
- **순차 호출 사이에 공유 가변 상태를 두지 마세요.** 재시도 시 이전 응답이 섞입니다.
- **SOAP envelope을 문자열 결합으로 만들지 마세요.** 빌더와 파서를 고정하고 타임아웃을 REST와 분리하세요.
- **`import()` 캐시를 무효화 없이 재사용하지 마세요.** 핫 리로드는 쿼리 문자열 변경 등 명시적 무효화가 필요합니다.
- **`ctx` 외부의 `fetch`를 직접 호출하지 마세요.** 타임아웃과 로그 상관관계가 깨집니다.

## Verify

- `plugins/<feature>/plugin.json`의 `route`가 Host 기동 로그에 등록 로그로 남는지 확인하세요.
- 플러그인 폴더 삭제 후 해당 `route`가 `404`를 반환하는지 확인하세요.
- 외부 API 장애 시 `handle` 타임아웃이 Host의 에러 응답으로 변환되는지 확인하세요.

## Pointers

- VS Code 시작 흐름과 확장 호스트 구조가 필요하면 `analysis/vscode/vscode_analyze.md`를 읽으세요.
- 매니페스트와 빌드 체계가 필요하면 `analysis/vscode/vscode-package.json-analyze.md`를 읽으세요.
