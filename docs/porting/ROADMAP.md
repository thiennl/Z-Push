# Roadmap: Port Z-Push (PHP) -> Kotlin, tu dau den go-live very high traffic

Gia dinh: 2-3 ky su + AI-assisted. Thoi gian la uoc luong, can hieu chinh sau Phase 0.
Muc tieu: dung hieu nang (long-poll quy mo lon, state nhanh), dung chi phi token (port theo module, golden test, codegen).

## 0. Nguyen tac kien truc (khoa tu Phase 0)

| Quyet dinh | Lua chon toi uu | Ly do |
|---|---|---|
| Runtime | JDK 25 LTS, generational ZGC, compact object headers, AppCDS | GC pause thap, RAM/object thap, khoi dong nhanh |
| HTTP | Ktor + Netty (hoac Vert.x) | Non-blocking, streaming body WBXML |
| Concurrency | Coroutines cho I/O; thu vien blocking (Jakarta Mail, ical4j, LDAP) chay tren dispatcher virtual-thread | Long-poll ton ~KB/ket noi, khong dung nhieu pool |
| Long-poll | `withTimeout` + Flow/Channel + dispatcher notify; KHONG sleep-poll | Thay `sleep()` trong synccollections.php |
| Push/change detect | IMAP IDLE gop theo user (multiplex), CalDAV/CardDAV sync-token/ctag, Redis pub/sub tu MTA/Dovecot | Giam polling backend |
| State | Postgres (state ben) + Redis (hot cache, lock, pubsub); `kotlinx.serialization` (protobuf/CBOR) | Bo serialize() PHP + file; cluster-friendly |
| Lock per device | Redis lock / pg advisory lock, lease + fencing | Thay IPC shared mem/memcached |
| Auth | Cache ket qua auth theo TTL ngan, rate-limit theo IP/user | Giam tai backend auth |
| Observability | OpenTelemetry + Micrometer/Prometheus, structured log (khong log WBXML day du o prod) | Bat buoc cho canary/rollback |
| Build | Gradle Kotlin DSL, version catalog, detekt + ktlint | On dinh cho AI sinh code |

## 1. Phan loai pham vi (Phase 0, 2-3 tuan)

- Chot backend can port: uu tien IMAP (10k LOC) + CalDAV/CardDAV (4k). **Kopano (22.9k, MAPI) khong port bang LLM**: goi qua REST/gRPC/IMAP hoac bo.
- Chot ban ActiveSync can ho tro (doi chieu `zpushdefs.php`/config; thuong 2.5, 12.x, 14.x, 16.x phan nua) va danh sach client (iOS, Android, Outlook, Samsung).
- Thu thap **traffic that**: ghi WBXML request/response (da an danh) theo lenh, version, thiet bi -> **golden corpus**.
- Do baseline PHP: p50/p95/p99 theo lenh, so thiet bi dong thoi, RAM/worker, ty le Ping/Sync long-poll.
- Danh gia rui ro: recurrence/timezone, meeting request, S/MIME, EAS rights management, provisioning/remote wipe.
- Deliverable: ADRs, scope matrix tinh nang x backend x version, danh sach rui ro, SLO (vd: p99 Sync < 800ms, 50k thiet bi/long-poll/node).

## 2. Nen tang (Phase 1, 3-4 tuan)

1. Skeleton Gradle multi-module: `wbxml`, `protocol`, `syncobjects`, `state`, `core`, `backend-api`, `backend-imap`, `backend-dav`, `server`, `tools`.
2. **Codegen** (script, khong LLM): `wbxmldefs.php` -> Kotlin tables; `syncobjects/*.php` -> data class + ma anh xa tag.
3. WBXML codec streaming (decoder/encoder, multipart, opaque/base64); fuzz + golden byte-by-byte.
4. Harness **golden replay**: dua request tu corpus qua server Kotlin va so response (normalize SyncKey/timestamp).
5. CI: build, detekt/ktlint, unit, golden, JMH cho WBXML, Testcontainers (Postgres, Redis, Dovecot, Radicale/SabreDAV).
- Exit: codec pass 100% golden, throughput WBXML >= 5x PHP (do JMH).

## 3. Core protocol + state (Phase 2, 4-6 tuan)

1. Request pipeline: auth, device identify, ban EAS, header parsing, error codes/status map.
2. State machine (`IStateMachine`, 10 phuong thuc) tren Postgres + Redis; SyncKey/counter, hierarchy cache, `ChangesMemoryWrapper` thanh cau truc co chi doc.
3. Lenh trung tam: `Provision`, `FolderSync`, `GetItemEstimate`, `Sync` (client->server, server->client, conflict, MoreAvailable, windowing), `Ping`.
4. Long-poll engine: heartbeat clamp, hang doi thay doi, huy khi client ngat, backpressure.
5. Device manager: policy, wipe, loop detection (thiet ke lai bang counter co han, khong port 1:1 `loopdetection.php`).
- Exit: FolderSync/Sync/Ping chay voi backend gia lap; soak 24h, khong ro ri coroutine.

## 4. Mail IMAP/SMTP (Phase 3, 6-8 tuan)

- `backend-imap` bang Jakarta Mail (Angus) tren virtual threads; pool ket noi theo user, IDLE multiplex, UID/MODSEQ (CONDSTORE/QRESYNC neu server ho tro) thay full diff.
- Parse MIME/chuyen body (plain/HTML/MIME), attachment stream, `ItemOperations`, `GetAttachment`, `SendMail`/`SmartReply`/`SmartForward`, `MoveItems`, `Search`, flags, OOF.
- Toi uu: tranh tai body khi chi can header; cache BODYSTRUCTURE; stream thang vao encoder WBXML.
- Exit: golden mail 100%, test tuong thich iOS/Android/Outlook that.

## 5. Contacts, Calendar, Tasks, Notes (Phase 4, 6-8 tuan)

- `backend-dav` voi Ktor client / Jackrabbit-light; sync-token/ctag; ical4j (recurrence, VTIMEZONE), ez-vcard.
- Bang chuyen **timezone** EAS<->IANA (thay `timezoneutil.php`), recurrence, exceptions, meeting request/response, attendee.
- Test property-based cho recurrence (sinh lich ngau nhien, so voi PHP oracle).
- Exit: round-trip khong mat truong, khop golden voi PHP tren corpus lich.

## 6. Tinh nang con lai (Phase 5, 3-4 tuan)

`ResolveRecipients`, `Settings`, `Find`, `ValidateCert`, `MeetingResponse`, GAL qua LDAP (UnboundID), Autodiscover, admin/CLI thay `z-push-admin`/`z-push-top`, webservice quan tri.

## 7. Hieu nang va do ben (Phase 6, 3-4 tuan, chay song song Phase 4-5)

- Load test voi simulator ActiveSync (Gatling/k6 + replay corpus): muc tieu >= 2-3x thiet bi dong thoi/node so voi PHP-FPM.
- Tinh chinh: JFR + async-profiler, ZGC, kich thuoc heap, Netty buffer pooling (zero-copy), batch ghi state, nen state (zstd), chong thundering herd khi mat ket noi hang loat.
- Chaos: mat Redis/Postgres/IMAP, failover, restart rolling, mang cham.
- Bao mat: audit auth, TLS, SSRF qua CalDAV URL, gioi han kich thuoc body/WBXML (giu cap vong lap kieu `MAXLOOP`), dependency scan.

## 8. Cutover va go-live (Phase 7, 4-6 tuan)

1. **Shadow mode**: sao chep traffic that sang Kotlin (khong tra ve client), so response, do lech.
2. **Canary theo nhom nguoi dung** (1% -> 5% -> 25% -> 50% -> 100%) qua LB/header; kill-switch tuc thi.
3. State: chon mot trong hai (quyet dinh o Phase 0):
   - A (khuyen nghi): khong migrate state PHP; client resync (SyncKey 0). Gioi han toc do resync, lam theo dot de tranh dinh tai.
   - B: importer doc state PHP serialized (ton cong, chi lam neu resync khong chap nhan duoc).
4. Runbook, alert (error rate theo lenh, latency, so long-poll, GC pause, backlog IDLE), on-call.
5. Giu PHP o che do chi-doc/fallback 2-4 tuan sau 100%.
- Exit go-live: 2 tuan lien tuc dat SLO o 100%, khong su co P1.

## 9. Moc thoi gian tong (uoc luong)

Phase 0: 2-3 | 1: 3-4 | 2: 4-6 | 3: 6-8 | 4: 6-8 (chong lan) | 5: 3-4 | 6: song song | 7: 4-6
=> ~6-8 thang den 100% traffic, 4-5 thang den canary dau tien (Mail + Contacts/Calendar co ban).

## 10. Rui ro chinh

| Rui ro | Giam thieu |
|---|---|
| Lech hanh vi so voi PHP (recurrence/TZ) | Oracle PHP + property-based + golden calendar |
| Client la (iOS/Outlook) loi sau cutover | Shadow + canary theo loai thiet bi |
| Dinh tai khi resync hang loat | Rate-limit resync, dot nho, tang batch dan |
| Vo ngan sach token | Skill port, codegen, golden-first, port theo module |
| Backend IMAP gioi han ket noi | Multiplex IDLE, pool theo user, circuit breaker |
