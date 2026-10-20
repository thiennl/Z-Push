---
name: port-zpush-kotlin
description: Port mot module/tinh nang cua Z-Push (PHP, ActiveSync) sang Kotlin (JDK 25, Ktor, coroutines) theo quy trinh tiet kiem token, golden-test-first, san sang very high traffic. Dung khi nguoi dung yeu cau port, viet lai, hoac review ban port Z-Push sang Kotlin.
---

# Port Z-Push -> Kotlin

Tham chieu: `docs/porting/ROADMAP.md` (kien truc va thu tu phase). Doc truoc khi bat dau.

## Quy trinh moi module (bat buoc theo thu tu)

1. **Xac dinh pham vi**: chi doc file PHP cua module va interface lien quan (`src/lib/interface/*`). Khong nap ca repo; bo qua `src/vendor`, `src/include/*` (thay bang thu vien), backend `ipc*`, backend Kopano, code compat PHP cu.
2. **Spec ngan** (<= 40 dong): input, output, trang thai, loi, side effect, cac nhanh dac biet. Luu `docs/porting/specs/<module>.md`.
3. **Golden test truoc**: dung fixture trong `golden/` (request/response WBXML da an danh). Neu thieu, tao fixture bang script chay ban PHP. Khong viet code khi chua co test do.
4. **Codegen truoc LLM**: bang hang so, tag WBXML, data class `syncobjects` sinh bang script o `tools/codegen`. Khong dung LLM cho phan bang thuan.
5. **Viet Kotlin toi thieu** de pass test, roi refactor.
6. **Chay**: `./gradlew :<module>:test detekt ktlintCheck` va golden replay. Sua theo loi cu the, khong viet lai ca file.
7. **Ghi chu lech hanh vi** co chu y (PHP quirk giu lai hay bo) vao spec.

## Quy uoc Kotlin (de code ngan, de LLM sinh dung)

- `data class` + `sealed interface` thay getter/setter va hierarchy cua `SyncObject`.
- Loi: `sealed class SyncResult`/`Result`, exception chi cho loi khong phuc hoi duoc. Map PHP exception -> ma loi EAS trong mot cho (`StatusMapper`).
- Concurrency: `suspend` + `Flow`; thu vien blocking goi qua `withContext(virtualThreadDispatcher)`. Cam `Thread.sleep`, `runBlocking` trong request path.
- Long-poll: `withTimeout(heartbeat)` + `Channel/SharedFlow` thong bao thay doi; huy khi client ngat (cooperative cancellation).
- WBXML: streaming, khong dung chuoi trung gian; ghi thang vao `ByteWriteChannel`; dung `ByteArray` pool, khong `String` cho payload lon.
- State: interface `StateStore` (Postgres + Redis), serialize bang `kotlinx.serialization` protobuf/CBOR; khong dung Java serialization.
- Immutable + `persistentList` o noi can chia se; khong global mutable state.
- Moi coroutine/ket noi/ban ghi cache phai co gioi han (timeout, kich thuoc, TTL).
- Log co cau truc, khong log noi dung email/WBXML day du o INFO; correlation id theo request.

## Anh xa nhanh PHP -> Kotlin

| PHP | Kotlin |
|---|---|
| `sleep()` trong vong lap Ping/Sync | `withTimeout` + `Flow` su kien |
| `serialize()`/file state | `StateStore` + kotlinx.serialization |
| IPC shared mem/memcached | Redis lock/pubsub |
| `ChangesMemoryWrapper` | immutable snapshot + diff |
| `InWhile()`/`MAXLOOP` | gioi han kich thuoc/do sau o decoder |
| `SyncObject` getter/setter | `data class` (codegen) |
| `*Exception` | `sealed` result + `StatusMapper` |

## Dinh nghia hoan thanh (Definition of Done)

- Golden test cua module pass 100%, khong flaky.
- detekt + ktlint sach; khong `!!` ngoai test.
- Co benchmark JMH cho hot path (WBXML, diff, parse) so voi baseline.
- Co spec + ghi chu lech hanh vi; metric/log da gan (latency, loi, kich thuoc).
- Khong lam tang so luong thread/ket noi theo so thiet bi (kiem tra bang soak test).

## Tiet kiem token

- Mot module/lan; chi dua file PHP can port + interface + test fail vao context.
- Khi sua loi, gui diff + thong bao loi, khong dan lai ca file.
- Tai su dung spec thay vi mo ta lai; cat comment header/licence khi dua PHP vao context.
- Uu tien viet script sinh code cho moi thu lap lai >= 10 lan.

## Khong lam

- Port backend Kopano/MAPI bang LLM (dung REST/gRPC/IMAP/DAV).
- Port 1:1 `loopdetection.php`, `ipc*`, PEAR (`include/`): thiet ke lai.
- Tu y doi hanh vi protocol de "dep hon" ma khong co golden/oracle chung minh.
