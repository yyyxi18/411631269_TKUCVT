# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：Layers 是 image 的唯讀檔案系統分層。每一層都代表一次檔案系統變更，例如安裝套件、複製程式碼、建立檔案。多個 image 可以共用相同 layer，所以 Docker 不需要把一樣的底層內容重複存很多次。
- Config 是什麼：Config 是 image 的執行設定，裡面會記錄 `Cmd`、`Entrypoint`、`Env`、`WorkingDir`、`User`、`ExposedPorts` 等 metadata。容器啟動時，Docker 就是照這份 config 決定怎麼跑。
- Manifest 是什麼：Manifest 是 image 的總索引，負責把 config 和所有 layers 串起來，記錄每個 layer 的 digest、大小與順序。Registry 也是靠 manifest 才知道要下載哪些 layer。

## python:3.12-slim inspect 摘錄
- Config.Cmd：`["python3"]`
- Config.Env：`PATH=/usr/local/bin:$PATH`、`LANG=C.UTF-8`、`GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305`、`PYTHON_VERSION=3.12.13`、`PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684`
- Config.WorkingDir：空字串（官方 Dockerfile 沒有另外設定 `WORKDIR`）
- RootFS.Layers 數量：（填入你 `docker image inspect python:3.12-slim` 的實測值）

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 24.320s |
| v1 改 app.py 後 rebuild | 4.953s |
| v2 首次 build | 11.188s |
| v2 改 app.py 後 rebuild | 1.005s |

觀察（用自己的話寫）：  
`Dockerfile.v2` 把 `requirements.txt` 單獨複製並先安裝相依，所以只改 `app.py` 時，`COPY requirements.txt` 和 `RUN pip install -r requirements.txt` 這兩層的 cache key 不會變，Docker 可以直接重用快取。相反地，`Dockerfile.v1` 先 `COPY app/ .`，只要 app 目錄任何檔案變動，這層就會 cache miss，後面的 `pip install` 也會一起重跑，所以 rebuild 幾乎和第一次 build 一樣慢。

## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | `argv = ['show_args.py', 'default1', 'default2']`，`PID = 7` | 錯誤：`exec: "extra1": executable file not found in $PATH` |
| CMD exec form | `argv = ['show_args.py', 'default1', 'default2']`，`PID = 1` | 錯誤：`exec: "extra1": executable file not found in $PATH` |
| ENTRYPOINT + CMD | `argv = ['show_args.py', 'default1', 'default2']`，`PID = 1` | `argv = ['show_args.py', 'extra1', 'extra2']`，`PID = 1` |

結論（用自己的話寫）：  
這次實驗可以看出，只有 `ENTRYPOINT + CMD` 這種組合會把 `docker run` 後面加的參數附加到主程式後面，因此最適合「固定跑某個程式，但允許使用者覆蓋預設參數」的情境。`CMD` 不管是 shell form 還是 exec form，只要在 `docker run` 後面加參數，原本的命令都會被整條覆蓋，所以容器會嘗試直接執行 `extra1`，最後報錯。另一方面，shell form 的 PID 不是 1，而是由 shell 再啟動 Python；exec form 和 ENTRYPOINT 版本的 PID 1 都直接是 Python，這也是為什麼正式環境通常更偏好 exec form。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.62GB |
| python:3.12-slim（runtime base） | 179MB |
| myapp:v2（單階段） | 197MB |
| myapp:multi（多階段） | 184MB |

解釋（用自己的話寫）：  
multi-stage build 把建置時需要的完整 Python 環境、安裝流程與中間產物留在 builder stage，最後只把 runtime 真正需要的檔案複製到 `python:3.12-slim`。所以最終 `myapp:multi` 比單階段的 `myapp:v2` 小，因為它沒有把整個 builder base 一起帶進去。從這次結果也看得出，builder base `python:3.12` 有 1.62GB，但最終 `myapp:multi` 只有 184MB，代表 builder stage 的 layer 沒有進最終 image，而是只留在本機 cache 供下次 build 使用。

## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 44K | 151M | 48K |
| build context 傳輸大小 | 未量測 | 129B | 129B |
| build 時間 | 未量測 | 1.092s | 1.155s |

觀察：這次故障注入後，專案目錄從 44K 增加到 151M，但 `docker build` 的 `transferring context` 仍然只有 129B，加上 `.dockerignore` 後也沒有明顯變化。原因不是 `.dockerignore` 沒作用，而是這次使用的 BuildKit 和 `Dockerfile.multi` 只需要 `app/requirements.txt` 與 `app/`，所以多餘的 `.git` 和 `logs` 本來就沒有被送進實際 build context。換句話說，這次量到的是「BuildKit 最小化 context」的效果，而不是傳統 builder 把整個目錄全送進去的情況。

## 排錯紀錄
- 症狀：我在專案目錄中加入 150MB 的垃圾檔後，原本以為 `docker build` 的 build context 會大幅增加，但實測仍然只有 129B。
- 診斷：檢查 build 輸出後發現使用的是 BuildKit，而且 `Dockerfile.multi` 只會 `COPY` `app/requirements.txt` 和 `app/`。因此 BuildKit 只傳送實際需要的檔案，沒有把 `.git` 和 `logs` 一起送進去。
- 修正：加入 `.dockerignore` 仍然是正確做法，因為它可以避免在其他 Dockerfile 或非 BuildKit 情境下把垃圾檔與敏感檔送進 image；只是這次範例剛好已經被 BuildKit 最小化了。
- 驗證：即使目錄大小變成 151M，build 輸出中的 `transferring context` 仍為 129B，證明額外垃圾檔沒有進入這次的 build context。

## 設計決策
我在 runtime stage 選擇 `python:3.12-slim`，而不是 `python:3.12-alpine`。原因是 `slim` 已經比完整版小很多，同時仍然使用 Debian 生態，對大多數 Python 套件的相容性比較好。`alpine` 雖然更小，但它使用的是 musl libc，不是常見的 glibc，很多 Python 套件的預編譯 wheel 可能不能直接用，常常會變成還要自己編譯，反而增加建置複雜度。對這次作業這種 Flask 範例來說，`slim` 是比較平衡的選擇。