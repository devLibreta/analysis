# VS Code 시작 흐름

## 전체 ASCII 트리

```text
OS (바이너리 실행)
└─ Electron 메인 프로세스 생성 (C++)   ★ 최초·유일 main, 아래 ★는 자식 OS프로세스 생성
   ├─ [mainImpl.ts] 인자·userData·메뉴·크래시·스킴 설정 + app.once('ready') 등록
   │   └─ ready 방출 = main 초기화 완료 (BrowserWindow 아님)
   └─ [JS] onReady() → startup()
      └─ bootstrapESM → import electron-main/main.js
         └─ CodeMain.main() → startup()
            ├─ createServices / initServices (경로 mkdir·state·config)
            ├─ claimInstance (mainIPCHandle 선점, 실패 시 기존 인스턴스에 위임 후 종료)
            └─ CodeApplication.startup()
               ├─ setupSharedProcess (객체 등록만, spawn 아님)
               ├─ initServices (Windows·Dialog·Launch·Storage·Terminal·Telemetry 등 ~20종)
               ├─ initChannels·프로토콜 핸들러
               └─ openFirstWindow      ★ 렌더러 생성 (window 마다 BrowserWindow 1개 = 별도 OS프로세스)
                  ├─ workbench 로드 (렌더러 내부, 요청자)
                  ├─ 요청:렌더러→실행:main Shared: SharedProcess + UtilityProcess → SharedProcessMain (싱글톤, 지연)
                  ├─ 요청:렌더러→실행:main extensionHost: ExtensionHostStarter + WindowUtilityProcess → ExtensionHostMain (window 마다 생성, 지연)
                  ├─ 요청:렌더러→실행:main ptyHost: PtyHostStarter + UtilityProcess → startPtyHost + PtyService (싱글톤, 지연)
                  └─ 요청:렌더러→실행:main fileWatcher: UtilityProcessWorker + WindowUtilityProcess → UniversalWatcher (window 마다 생성, utilityWorker 경유)
```

## 렌더러 프로세스 생성까지

```mermaid
flowchart LR

   %% node %%
   USER[사용자]
   APP[VSCODE]
   subgraph "메인 프로세스 (단일 OS프로세스)"
      
      subgraph "일렉트론 프레임워크(Electron C++)"
         E_ST[초기화 시작]
         E_ING[초기화 재개]
         E_FNS["초기화 완료<br>(메시지루프 진입)"]
         
      end
      subgraph "노드 소스(내장 런타임)"
         N_ST[로직 시작]
         N_ENTRY[로직 Entry]
         N_READY["app.once('ready')"]
         N_FNS[로직 종료]
         N_CALLBACK["onReady()"]
         N_START["startup()"]

      end
   end

   subgraph "렌더러 프로세스(Chromium)"
      R_PROC["BrowserWindow"]
   end
   

   %% process %%
   USER --> |바이너리 실행| APP

   APP --> |메인 프로세스 시작| E_ST
   
   E_ST--> |"JS진입<br>부트스트랩"| N_ST
   N_ST--> N_ENTRY
   N_ENTRY --> |ready 이벤트 리스너 및 콜백 등록| N_READY
   N_READY --> N_FNS
   N_FNS --> |"제어권 반환<br>이벤트루프 복귀"| E_ING
   E_ING --> E_FNS
   E_FNS --> |"ready 이벤트 발행<br>(=등록 콜백 호출)"| N_CALLBACK
   N_CALLBACK --> N_START

   N_START --> |"window 마다 생성"| R_PROC

   %% style %%
```

## 렌더러 생성 이후

* 싱글톤인 자식 프로세스는 한번 생성 후는 연결만 가져온다.

```mermaid
flowchart LR

   %% node %%

   subgraph "렌더러 프로세스(Chromium)"
      R_PROC["BrowserWindow"]
   end

   subgraph "메인 프로세스 (단일 OS프로세스)"
      SH_ST["SharedProcess"]
      SH_UP["UtilityProcess"]
      EX_ST["ExtensionHostStarter"]
      EX_UP["WindowUtilityProcess"]
      PTY_ST["PtyHostStarter"]
      PTY_UP["UtilityProcess"]
      WK_ST["UtilityProcessWorker"]
      WK_UP["WindowUtilityProcess"]
   end

   subgraph "자식 프로세스"
      SH["SharedProcessMain<br>(싱글톤)"]
      EX["ExtensionHostMain<br>(window 마다 생성)"]
      PTY["startPtyHost + PtyService<br>(싱글톤)"]
      FW["UniversalWatcher<br>(window 마다 생성)"]
   end

   %% process - 요청:렌더러→실행:main %%
   R_PROC --> |"프로세스 생성 요청<br>(IPC)"| SH_ST
   R_PROC --> |"프로세스 생성 요청<br>(IPC)"| EX_ST
   R_PROC --> |"프로세스 생성 요청<br>(IPC)"| PTY_ST
   R_PROC --> |"프로세스 생성 요청<br>(IPC)"| WK_ST
   SH_ST  --> |"호출"| SH_UP
   EX_ST  --> |"호출"| EX_UP
   PTY_ST --> |"호출"| PTY_UP
   WK_ST  --> |"호출"| WK_UP
   SH_UP  --> |"fork<br>lazy spawn"| SH
   EX_UP  --> |"fork<br>lazy spawn"| EX
   PTY_UP --> |"fork<br>lazy spawn"| PTY
   WK_UP  --> |"fork<br>lazy spawn"| FW

   %% style %%
   classDef nowrap white-space:nowrap
   class R_PROC,SH_ST,SH_UP,EX_ST,EX_UP,PTY_ST,PTY_UP,WK_ST,WK_UP,SH,EX,PTY,FW nowrap
```

## 확장호스트 내부

* 확장호스트 프로세스는 자체 기능 없음. 껍데기(`ExtensionHostMain` + `RPC`) + 로드된 확장 코드(`ExtHost*`) + 렌더러의 `MainThread*` 조합으로 동작한다.

```mermaid
flowchart LR

   %% node %%

   subgraph "자식 프로세스 (extensionHost)"
      EX_MAIN["ExtensionHostMain"]
      EX_RPC["RPCProtocol<br>자식 측"]
      EX_SVC["ExtHostExtensionService"]
      EXT["확장 코드<br>ExtHost* 구현"]
   end

   subgraph "렌더러 프로세스 (Chromium)"
      R_PROTO["RPCProtocol<br>렌더러 측"]
      R_MGR["ExtensionHostManager<br>MainThread* 구현"]
   end

   %% process %%
   EX_MAIN --> |"new RPCProtocol"| EX_RPC
   EX_MAIN --> |"initialize()"| EX_SVC
   EX_SVC --> |"확장 JS 로드"| EXT
   EXT --> |"ExtHost* 등록"| EX_RPC
   EX_RPC --> |"MessagePort/Socket<br>IPC"| R_PROTO
   R_PROTO --> |"MainThread* 호출"| R_MGR
   R_MGR --> |"ExtHost* 프록시 호출"| R_PROTO

   %% style %%
   classDef nowrap white-space:nowrap
   class EX_MAIN,EX_RPC,EX_SVC,EXT,R_PROTO,R_MGR nowrap
```
