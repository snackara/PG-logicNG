# Báo cáo xác nhận tích hợp LogicNG/Plaisted–Greenbaum vào StateLocate

## 1. Kết luận ngắn

Kiến trúc trong hai hình đầu **có thể triển khai được**, với một điều chỉnh quan trọng:

* Python vẫn parse input hiện tại (kể cả Python IR hoặc Verilog nếu project hiện tại đã hỗ trợ), rồi gửi **Boolean IR trung gian có schema/version rõ ràng** sang một Java sidecar chạy lâu dài.
* Sidecar dùng **một `FormulaFactory` cho cả một session**, dựng công thức LogicNG từ IR, tạo công thức base và các công thức guarded cycle `activation_i => cycle_i`.
* Để hai backend nhận **đúng cùng một CNF**, phải dùng CNF hiện hữu (materialized CNF) qua `CNFEncoder` với `CNFConfig.Algorithm.PLAISTED_GREENBAUM`, tách thành clause stream một lần, rồi nạp chính stream đó vào LogicNG MiniSat và Z3. Không dùng `PG_ON_SOLVER` cho đường benchmark này, vì CNF và auxiliary variable khi ấy chỉ tồn tại bên trong solver.
* Cấu hình `atomBoundary(0)` nếu yêu cầu “Plaisted–Greenbaum cho mọi công thức không-CNF”. Mặc định `atomBoundary = 12`; công thức NNF có ít hơn 12 atom được **factorization**, không phải PG.
* LogicNG không làm mất literal một cách tùy tiện. Tuy nhiên FormulaFactory, NNF, factorization/PG và solver đều có các phép chuẩn hóa hợp lệ có thể khiến literal/clauses không còn xuất hiện đúng như input. Cam kết đúng phải là **equisatisfiable và bảo toàn nghiệm trên source variables**, không phải bảo toàn cú pháp hay buộc clause cycle phải chứa literal của base.
* Liên kết base–cycle được đảm bảo bằng **cùng tên/ID cho cùng source variable trong một global symbol table**, không phải bằng auxiliary variable và cũng không cần một clause cycle giao cú pháp với một clause base.
* LogicNG 2.6.2 có solver Java tích hợp: MiniSat 2.2, Glucose 2 và MiniCard. Với yêu cầu incremental, assumptions và save/load state, lựa chọn phù hợp nhất là `MiniSat.miniSat(...)` với `incremental(true)`.

## 2. Phạm vi và phiên bản đã xác nhận

Tài liệu này bám theo đúng source tree hiện tại, commit `93bc457bb6dc29e658a6f8c1b836cec6ed2a4b62` (“Final 2.6.2 release”), không suy diễn API của một major version khác.

| Mục | Giá trị đã xác nhận |
|---|---|
| Maven coordinates | `org.logicng:logicng:2.6.2` |
| Java source/target | Java 8 |
| License | Apache License 2.0 |
| PG public transformation | `org.logicng.transformations.cnf.PlaistedGreenbaumTransformation` |
| CNF facade | `org.logicng.transformations.cnf.CNFEncoder` / `CNFConfig` |
| SAT facade | `org.logicng.solvers.SATSolver`, implementation `MiniSat` |
| Solver cores | MiniSat 2.2, Glucose 2, MiniCard |
| Trạng thái release | `README.md` khuyến nghị nhánh release cho production; pin artifact và checksum/commit, không dùng snapshot |

Dependency nên pin cứng:

```xml
<dependency>
  <groupId>org.logicng</groupId>
  <artifactId>logicng</artifactId>
  <version>2.6.2</version>
</dependency>
```

Khi đóng gói sidecar, cần giữ `LICENSE` và notice của các dependency theo quy trình compliance của project. LogicNG dùng Apache-2.0, cho phép dùng thương mại và sửa đổi với các nghĩa vụ thông báo/license tương ứng.

> Lưu ý migration: tài liệu này cố ý không dùng API của LogicNG thế hệ/package khác. Nếu đổi major version, phải chạy lại toàn bộ contract test ở mục 14 trước khi thay dependency.

## 3. LogicNG thực sự nhận đầu vào và trả đầu ra gì?

### 3.1. Input

LogicNG là thư viện Java thao tác trên object graph `Formula`, tạo bởi `FormulaFactory`. Input lõi không phải file Python và cũng không phải Verilog. Có ba cách liên quan trong tree này:

1. Dựng object trực tiếp: `variable`, `literal`, `and`, `or`, `not`, `implication`, `equivalence`, cardinality/PB constraint.
2. Đọc **DIMACS CNF** bằng `DimacsReader`.
3. Source tree có grammar/parser propositional và pseudo-Boolean phục vụ test, nhưng ở release tree này ANTLR nằm dưới `src/test` và dependency ANTLR là test scope. Vì vậy sidecar production **không nên phụ thuộc** vào `PropositionalParser` như một API artifact được bảo đảm.

Không có Python-file parser hoặc Verilog parser production trong module này. Do đó:

* Không gửi source `.py` và kỳ vọng LogicNG tự hiểu BooleanIR.
* Không gửi `.v/.sv` và kỳ vọng LogicNG tự elaborate Verilog.
* Python/StateLocate chịu trách nhiệm parse/elaborate theo semantics hiện tại, sau đó serialize một Boolean IR trung lập.

### 3.2. Protocol IR đề xuất

Dùng JSON Lines hoặc length-prefixed JSON/MessagePack trên stdin/stdout của persistent JVM. JSON Lines dễ debug trước; chuyển sang protobuf chỉ khi profiling chứng minh protocol là bottleneck.

Ví dụ request:

```json
{"v":1,"id":"s-42","op":"open","factory":"run-42"}
{"v":1,"id":"b-1","op":"encode_base","expr":{"op":"and","args":[{"var":"state.ready"},{"op":"or","args":[{"op":"not","arg":{"var":"state.err"}},{"var":"input.retry"}]}]}}
{"v":1,"id":"c-7","op":"add_cycle","cycle":7,"activation":"@app$act$7","expr":{"op":"or","args":[{"op":"not","arg":{"var":"state.ready"}},{"var":"next.done"}]}}
{"v":1,"id":"q-7","op":"solve_incremental","assume_true":["@app$act$7"],"assume_false":["@app$act$0","@app$act$1"]}
```

Schema phải định nghĩa tường minh:

* node: `const`, `var`, `not`, `and`, `or`, `impl`, `equiv` (và PB/CC nếu thật sự cần);
* tên source symbol là UTF-8 string ổn định, không dựa vào object id Python;
* thứ tự operand được canonicalize tại Python hoặc sidecar;
* session, request id, protocol version, cycle id;
* semantics của empty `and` (true), empty `or` (false), duplicate operand và complementary operand;
* giới hạn depth/size và lỗi validation.

Không cho source symbol bắt đầu bằng `@RESERVED`, vì LogicNG dùng các prefix nội bộ `@RESERVED_CNF_`, `@RESERVED_CC_`, `@RESERVED_PB_`. Activation nên nằm trong namespace của ứng dụng, ví dụ `@app$act$<session>$<cycle>`, đồng thời được đăng ký rõ là `ACTIVATION`, không suy ra loại chỉ từ prefix.

### 3.3. Output

Sidecar nên trả cả dữ liệu semantic lẫn thống kê:

```json
{
  "v": 1,
  "id": "c-7",
  "status": "OK",
  "encoding": "PLAISTED_GREENBAUM",
  "atom_boundary": 0,
  "clauses": [[-31,12,44],[31,-12],[31,-44]],
  "symbols": [
    {"id":12,"name":"state.ready","kind":"SOURCE"},
    {"id":31,"name":"@app$act$7","kind":"ACTIVATION"},
    {"id":44,"name":"@RESERVED_CNF_run_42_0","kind":"ENCODER_AUX"}
  ],
  "counts":{"source_variables":1,"activation_variables":1,"auxiliary_variables":1,"variables_total":3,"clauses":3,"literal_occurrences":7}
}
```

Đây là output của adapter, không phải một DTO có sẵn của LogicNG. `CNFEncoder.encode()` trả một `Formula` ở dạng CNF. Adapter duyệt CNF để tạo clause stream, mapping integer và counts.

## 4. “Optimization có làm mất literal không?”

### 4.1. Câu trả lời chính xác

Có thể literal/operand input **không xuất hiện trong CNF cuối**, nhưng không phải bị mất ngẫu nhiên:

* `FormulaFactory` hash-cons/canonicalize formula, loại operand trùng trong `and/or`, flatten toán tử cùng loại và mặc định simplify complementary operands (`x & ~x -> false`, `x | ~x -> true`).
* NNF đẩy phủ định xuống literal và khử implication/equivalence theo logic tương đương.
* Với `atomBoundary` mặc định, công thức nhỏ đi qua factorization.
* PG tạo một **equisatisfiable extension** với auxiliary variables và chỉ sinh chiều implication cần theo polarity. Nó không cam kết CNF tương đương trên toàn bộ tập biến kể cả auxiliary, cũng không bảo toàn cấu trúc/cú pháp input.
* Clause tautology, duplicate literal/clause và constant có thể bị factory chuẩn hóa.
* Khi vào solver, unit propagation, clause simplification, learnt clause và loại satisfied clause có thể làm “formula hiện trên solver” khác clause stream đã nạp.

Cam kết cần kiểm thử là, với source variables `X`:

```text
F(X)  iff  exists A . PG(F)(X, A)
```

trong đó `A` là encoder auxiliaries. Đừng kiểm bằng “mọi literal input phải còn trong output”. Ví dụ `x | ~x` hợp lệ khi trở thành `true`, nên `x` biến mất; `x & x` hợp lệ khi chỉ còn `x`.

### 4.2. Base và cycle có còn liên quan không?

Có, nếu chúng dùng cùng source symbol mapping. Giả sử:

```text
base  = ready & (grant => run)
cycle = run & next_ready
guarded_cycle = act_7 => cycle
```

CNF(base) và CNF(guarded_cycle) không cần có một clause giống nhau hay dùng chung auxiliary. Chúng liên hệ vì literal `run` có **cùng integer ID** ở cả hai clause set. Solver giải conjunction:

```text
CNF(base) AND CNF(act_7 => cycle)
```

và query cycle 7 dùng assumption `act_7`. Nếu `run` bị simplify khỏi một phần vì phần đó là hằng/tautology, thì phần đó thực sự không áp thêm ràng buộc lên `run`; đây là semantics đúng.

### 4.3. Điều kiện bắt buộc để không “đứt liên kết”

1. Một global `name -> integer ID` table cho cả session; ID không được tái sử dụng cho symbol khác.
2. Một `FormulaFactory` duy nhất cho base và mọi cycle trong session.
3. Không tạo lại/clear factory giữa base và cycle. Không trộn formula từ factory khác; mặc định merge strategy `PANIC` là tốt.
4. Không map theo thứ tự duyệt riêng từng CNF. Base và cycle phải lấy ID từ cùng symbol table.
5. Không dùng auxiliary variable làm public contract giữa Python và Java hoặc giữa các cycle.
6. Validate source name không va chạm namespace reserved/activation.
7. Giữ clause stream bất biến làm “ground truth” cho cả hai backend; hash stream để phát hiện lệch.

## 5. Cách dùng Plaisted–Greenbaum đúng và “đúng một lần”

### 5.1. Hai PG path khác nhau

LogicNG có hai cơ chế dễ bị nhầm:

| Cơ chế | Kết quả | Auxiliary ở đâu | Phù hợp benchmark cùng CNF? |
|---|---|---|---|
| `CNFEncoder` + `PLAISTED_GREENBAUM` | `Formula` CNF duyệt được | FormulaFactory, tên `@RESERVED_CNF_...` | **Có** |
| `MiniSatConfig.CNFMethod.PG_ON_SOLVER` | thêm thẳng vào core solver | solver-internal | Không, khó xuất cùng exact stream sang Z3 |
| `FULL_PG_ON_SOLVER` | thêm thẳng, không NNF trước (trừ PBC) | solver-internal | Không |

Đề xuất dùng path thứ nhất làm canonical encoder:

```java
FormulaFactoryConfig ffConfig = FormulaFactoryConfig.builder()
        .name("run_42")
        .formulaMergeStrategy(FormulaFactoryConfig.FormulaMergeStrategy.PANIC)
        .build();
FormulaFactory f = new FormulaFactory(ffConfig);

CNFConfig cnfConfig = CNFConfig.builder()
        .algorithm(CNFConfig.Algorithm.PLAISTED_GREENBAUM)
        .atomBoundary(0)
        .build();
CNFEncoder encoder = new CNFEncoder(f, cnfConfig);

Formula baseCnf = encoder.encode(base);
Formula cycleCnf = encoder.encode(f.implication(act7, cycle7));
```

`atomBoundary(0)` rất quan trọng: implementation thực hiện factorization khi `nnf.numberOfAtoms() < boundary`. Nếu để mặc định 12, nhãn telemetry “PG” không có nghĩa mọi input đều đã chạy PG.

“Đúng một lần” nên hiểu là:

* base được encode một lần;
* mỗi guarded cycle mới được encode một lần;
* không dựng lại `base AND all_cycles_so_far` rồi encode lại ở mỗi vòng;
* clause stream sinh ra được fan-out nguyên trạng cho LogicNG solver và Z3.

### 5.2. Polarity, NNF, cutoff/fallback và cache

* Public `PlaistedGreenbaumTransformation` luôn gọi `formula.nnf()` trước.
* Nếu input đã là CNF, nó trả CNF đó.
* Nếu số atom nhỏ hơn boundary, dùng factorization.
* Nếu thật sự PG, implementation sinh implication theo **positive polarity** của NNF rồi restrict top-level PG variable thành true.
* Kết quả/PG variable được cache trên formula. API transformation này luôn sử dụng cache nội bộ; tham số `cache` chỉ ảnh hưởng việc gắn entry của formula gốc với NNF entry.
* `CNFEncoder` giữ instance transformation và FormulaFactory hash-cons công thức, vì vậy cùng subformula trong cùng factory có thể tái dùng cache/auxiliary. Điều này đúng về semantics nhưng ảnh hưởng tên/số auxiliary và timing.
* `PLAISTED_GREENBAUM` thuần không có fallback Tseitin. Fallback chỉ liên quan `ADVANCED`; vì thế không chọn `ADVANCED` nếu contract yêu cầu PG.

### 5.3. Auxiliary provenance

Không có public result kiểu `(clauses, sourceVars, auxVars, provenanceBySubformula)`. Với explicit factory PG:

* auxiliary được đặt bởi `FormulaFactory.newCNFVariable()` với prefix reserved;
* transformation cache liên kết nội bộ subformula → PG variable nhưng không nên coi cache entry/prefix là protocol bền vững;
* sidecar phải tự gắn loại biến dựa trên tập đã đăng ký **trước encoding**.

Phân loại chắc chắn:

```text
SOURCE     = names do Python IR khai báo
ACTIVATION = names do sidecar cấp/đăng ký
ENCODER_AUX = variables(CNF) - SOURCE - ACTIVATION
```

Prefix chỉ dùng làm sanity check. Không dùng `startsWith("@RESERVED_CNF_")` làm nguồn sự thật duy nhất.

## 6. Trích xuất clause chính xác

LogicNG biểu diễn CNF có thể là constant/literal/clause/conjunction. Adapter phải xử lý đủ:

| CNF result | Clause stream |
|---|---|
| `$true` | 0 clause |
| `$false` | 1 empty clause `[]` |
| literal | 1 unit clause |
| `OR` các literal | 1 clause |
| `AND` | mỗi top-level operand là một clause |

Không được bỏ empty clause; đó là UNSAT. Tautological clause thường đã được factory rút thành true. `FormulaFactory.cnf(...)` dùng `LinkedHashSet`, nên duplicate top-level clause bị loại; `and/or` cũng loại duplicate operands. Việc này đúng logic nhưng counts là **counts sau normalization**.

ID đề xuất:

1. Khi nhận IR, đăng ký source variables theo thứ tự canonical (ví dụ lexicographic UTF-8 hoặc deterministic first occurrence).
2. Đăng ký activation IDs theo cycle index.
3. Sau encoding, đăng ký encoder aux mới theo thứ tự clause/literal traversal.
4. Literal DIMACS: `+id` cho phase true, `-id` cho phase false; ID bắt đầu từ 1.
5. Không dùng trực tiếp internal MiniSat encoding `2*index` / xor 1 làm wire format.

Các số liệu có thể cung cấp chính xác từ canonical clause stream:

```text
clauses              = số vector clause (kể cả empty)
literal_occurrences  = tổng độ dài các clause
variables_total      = số ID khác nhau (nên báo thêm allocated và referenced)
source_variables     = số source ID referenced/declared (báo cả hai nếu khác)
activation_variables = số activation ID referenced/declared
auxiliary_variables  = referenced variables - source - activation
max_clause_length, unit_clauses, binary_clauses, empty_clauses
```

`FormulaDimacsFileWriter` có thể ghi DIMACS và mapping, nhưng bridge trực tiếp nên trích xuất in-memory để tránh I/O và giữ mapping do adapter sở hữu. Writer chỉ nên dùng làm artifact debug/cross-check.

## 7. Tích hợp solver LogicNG

### 7.1. Fresh solver

Fresh solver là instance mới cho từng solve:

```java
MiniSatConfig solverConfig = MiniSatConfig.builder()
        .incremental(true)
        .cnfMethod(MiniSatConfig.CNFMethod.FACTORY_CNF)
        .build();

SATSolver fresh = MiniSat.miniSat(f, solverConfig);
fresh.add(baseCnf);
fresh.add(cycleCnf);
Tristate result = fresh.sat(Arrays.asList(act7));
Assignment model = result == Tristate.TRUE
        ? fresh.model(sourceVariables)
        : null;
```

Vì `baseCnf/cycleCnf` đã là CNF, đường add nhận CNF và nạp clause; không cần gọi solver-internal PG lần nữa. `FACTORY_CNF` làm ý định rõ hơn. Khi so với Z3, tốt hơn là adapter gọi helper nạp từng canonical clause thay vì để từng backend tự diễn giải Boolean IR.

### 7.2. Incremental solver

Một instance sống suốt session:

```java
SATSolver inc = MiniSat.miniSat(f, solverConfig);
inc.add(baseCnf);                         // permanent

inc.add(cycle0Cnf);                       // permanent guarded clauses
Tristate r0 = inc.sat(Arrays.asList(act0));

inc.add(cycle1Cnf);                       // append-only
Tristate r1 = inc.sat(Arrays.asList(act0.negate(), act1));
```

Assumption chỉ là conjunction tạm trong solve, không được thêm như unit clause permanent. Policy activation phải ghi rõ:

* **Chỉ cycle hiện tại:** assume `act_i=true` và tất cả activation cycle cũ `false` để model/semantics minh bạch.
* **Cumulative:** assume tất cả `act_0..act_i=true`.
* **Subset:** truyền vector true/false đầy đủ cho subset cần bật/tắt.

Nếu activation cũ không được assume false, solver có quyền chọn false nên thường không tạo false UNSAT, nhưng model và so sánh backend trở nên khó hiểu. Vì vậy nên truyền assignment activation đầy đủ.

### 7.3. Kết quả và model

`sat()` trả:

* `Tristate.TRUE`: SAT;
* `Tristate.FALSE`: UNSAT;
* `Tristate.UNDEF`: bị handler cancel/timeout, **không phải UNSAT**.

Chỉ gọi `model(...)` sau SAT. Truyền explicit source variable collection để projection không lẫn activation/auxiliary. Mặc định LogicNG cũng loại tên có reserved CNF/CC/PB prefix khỏi model, nhưng explicit projection an toàn hơn và không phụ thuộc prefix.

### 7.4. Reset, save/load và assumptions

* `reset()` xóa toàn bộ database và PG caches của solver; không dùng để chuyển cycle nếu muốn incremental.
* `saveState()/loadState()` chỉ quay ngược state của **cùng solver instance**; state chỉ lưu kích thước cấu trúc, không serialize để lưu file hay chuyển process.
* MiniSat và MiniCard hỗ trợ save/load khi incremental=true; Glucose ném `UnsupportedOperationException` cho save/load.
* Load một state cũ làm các state “tương lai” không còn hợp lệ.
* Public API cảnh báo assumption có side effect nếu nó giới thiệu variable chưa biết. Luôn add/register mọi activation trước khi solve. Nếu cần isolation tuyệt đối cho model enumeration/conflict và search history, save state trước nhóm assumption query rồi load lại sau.

## 8. Timeout, cancel và lifecycle

Ví dụ soft timeout cho mỗi `sat()`:

```java
TimeoutSATHandler timeout = new TimeoutSATHandler(
        2_000L,
        TimeoutHandler.TimerType.RESTARTING_TIMEOUT);
Tristate r = inc.sat(timeout, assumptions);
if (r == Tristate.UNDEF && timeout.aborted()) {
    // status = TIMEOUT, tuyệt đối không map sang UNSAT
}
```

Ba mode:

* `SINGLE_TIMEOUT`: deadline khởi tạo ở lần `started()` đầu; tái dùng handler không reset deadline.
* `RESTARTING_TIMEOUT`: deadline mới ở mỗi `started()`/solve; phù hợp per-call timeout.
* `FIXED_END`: tham số là epoch milliseconds tuyệt đối.

Đây là **soft/cooperative timeout**: handler được hỏi tại conflict, nên có thể quá hạn vài ms hoặc hơn nếu chưa đến callback. API này không cung cấp hard kill và không chứng minh thread-safe interrupt từ thread khác. Kiến trúc sidecar nên có hai tầng:

1. Soft timeout LogicNG trả `UNDEF`.
2. Python watchdog có hard deadline; nếu sidecar treo/quá grace period thì kill cả JVM process, đánh dấu request `KILLED`, khởi động sidecar mới và replay base/cycles từ canonical log.

Sau `UNDEF`, cách an toàn cho production là bỏ solver instance hiện tại và rebuild/replay, hoặc restore state snapshot đã lưu trước solve nếu đã được regression-test. Không mặc định coi database sau cancel là sạch tương đương fresh solver.

Protocol nên phân biệt: `SAT`, `UNSAT`, `TIMEOUT`, `CANCELED`, `ERROR`, `SIDECAR_DIED`; không gộp ba trạng thái cuối thành `UNKNOWN` nếu benchmark cần chẩn đoán.

## 9. Determinism và benchmark công bằng

### 9.1. Điều gì ổn định và điều gì không được hứa

FormulaFactory chủ yếu dùng interning/hash-cons và nhiều nơi dùng `LinkedHashSet`, nhưng mapping tên trong solver có `HashMap`; tên/ID auxiliary phụ thuộc lịch sử factory và thứ tự dựng/encode. Không có public guarantee rằng clause order, aux numbering hoặc search trace giống bit-for-bit giữa JDK/version/lịch sử cache khác nhau.

SAT/UNSAT phải ổn định; model cụ thể, runtime, learnt clauses, order và aux names có thể khác. JIT warm-up, GC và cache khiến cycle sau “nhanh giả” nếu so với fresh process.

### 9.2. Quy trình benchmark

* Pin LogicNG 2.6.2, exact JDK vendor/version, JVM flags, CPU affinity và memory limit.
* Canonicalize IR ordering; dùng cùng request log.
* Materialize một canonical clause stream và gửi byte-for-byte (cùng IDs/clauses) cho cả solver.
* Hash payload `(symbol table, clauses, assumptions)`; log hash ở cả backend.
* Tách thời gian: protocol, formula construction, PG encoding, solver load, solver solve, model projection.
* Warm up riêng; báo cả cold và steady-state.
* Fresh benchmark: instance/factory policy giống nhau giữa samples hoặc ghi rõ khác biệt.
* Incremental benchmark: cùng sequence add/assume; không so cycle thứ N incremental với cycle N fresh mà bỏ chi phí load ở một bên.
* Chạy nhiều iteration/process, báo median/p95 và timeout count.
* Không bật solver randomization không kiểm soát; LogicNG API 2.6.2 không lộ một seed thống nhất cho mọi core, vì vậy đừng tuyên bố reproducible search trace chỉ bằng “seed=...”.

## 10. FormulaFactory: xác nhận từng điểm

| Câu hỏi | Xác nhận/biện pháp |
|---|---|
| Simplification mặc định? | Có simplification cấu trúc: constants, flatten, duplicate; complementary operands mặc định được simplify. Có thể tắt riêng complementary simplification, nhưng không nên dựa vào factory để bảo toàn AST nguyên văn. |
| Canonicalization/interning? | Có; cùng formula trong cùng factory thường trả cùng object và cache transformation dùng chung. |
| Operand order? | Construction dùng `LinkedHashSet`, giữ insertion order sau dedup; commutative formula được intern. Dù vậy protocol phải tự canonicalize và không coi printed/order là ABI. |
| Variable name hỗ trợ ký tự nào? | API programmatic nhận string; để tránh parser/protocol ambiguity, dùng escaped JSON string và naming policy hạn chế. Cấm namespace `@RESERVED*`. |
| Trùng auxiliary prefix? | Có rủi ro nếu source tự dùng reserved prefix; reject input. Đặt factory name riêng session giảm clash giữa factory. |
| Factory khác nhau có dùng chung formula? | Mặc định `PANIC`; có `IMPORT` nhưng không nên dùng trong sidecar. |
| Thread-safe? | Không có contract thread-safe cho mutable factory/caches/solver. Một session/worker chỉ được một thread sở hữu; parallelism bằng nhiều isolated worker. |
| Dùng formula factory A với solver/factory B? | Không. Dùng cùng factory xuyên suốt; nếu bắt buộc phải import thì làm ở boundary có kiểm soát và test mapping. |

## 11. Clause extraction và solver statistics: giới hạn cần nói rõ

`FormulaOnSolverFunction` có thể reconstruct formula đang ở solver, nhưng documentation của nó nói rõ kết quả có thể khác input do CNF conversion, propagation, simplification, removed/learnt clauses; do đó **không dùng nó để thống kê encoder output**.

Core có `nVars()`, `clauses()`, `variables()` và internal data, nhưng đó không phải một public metrics contract đầy đủ (conflicts/decisions/propagations không có facade thống nhất, ổn định). Báo hai namespace metric tách biệt:

* `encoding.*`: tính trên immutable canonical clause stream — dùng để so variable/clause/aux.
* `solver.*`: backend-specific, chỉ xuất field nào public và đã versioned; không trộn learnt/simplified database với input counts.

DIMACS writer không nên quyết định global IDs cho production nếu adapter đã có mapping. Nếu xuất file debug, kèm `.map`, manifest version/hash và kiểm lại số header với stream.

## 12. Những giới hạn/ý không thể đáp ứng nguyên trạng

1. **LogicNG tự parse file Python/Verilog:** không thể với module này; cần parser/elaborator hiện có ở Python hoặc công cụ frontend khác.
2. **Bảo toàn mọi literal/clause đúng cú pháp input:** không thể đồng thời yêu cầu normalization/PG; chỉ bảo đảm projection/equisatisfiability.
3. **Dùng `PG_ON_SOLVER` nhưng lấy exact CNF sạch để cấp cho Z3:** không phải API phù hợp. Dùng explicit `CNFEncoder` hoặc tự instrument/fork core (không khuyến nghị).
4. **Stable public provenance subformula → auxiliary:** không có result API ổn định. Adapter chỉ nên phân loại source/activation/aux; nếu cần mapping subformula chi tiết phải duy trì encoder riêng/fork và đó là maintenance burden.
5. **Hard timeout chỉ bằng `TimeoutSATHandler`:** không thể; phải có process watchdog.
6. **Save/load solver qua process/file:** `SolverState` không phải serialization snapshot; replay canonical clauses khi restart.
7. **Glucose + save/load incremental state:** không hỗ trợ trong implementation này. Chọn MiniSat/MiniCard; với CNF thuần chọn MiniSat.
8. **UNSAT core chuẩn từ assumption list bằng đúng flow trên:** không nên cam kết nếu chưa thiết kế proof/proposition flow riêng. Proof generation có constraint với incremental/core; coi đây là feature riêng, không là output mặc định của SAT query.
9. **Deterministic clause/aux order xuyên mọi JDK/version:** không có guarantee public; adapter canonicalize output nếu cần reproducible artifact, nhưng canonicalization clause không được đổi symbol identity.

## 13. Kiến trúc sidecar đề xuất

```text
StateLocate/Python
  existing parser / Verilog frontend
       |
       v
  validated Boolean IR + source symbol table
       |
       | persistent framed protocol
       v
Java LogicNG sidecar (one owner thread per session)
  FormulaFactory(name=session, merge=PANIC)
  IR -> LogicNG Formula
  base; (act_i => cycle_i)
  CNFEncoder(PG, atomBoundary=0)
       |
       v
  canonical SymbolTable + immutable ClauseStore + per-part ranges
       |                              |
       v                              v
  LogicNG MiniSat                 Z3 adapter
  fresh / incremental             exact same clauses/IDs
```

ClauseStore nên lưu metadata mỗi batch:

```text
batch_id, kind(BASE|CYCLE), cycle_id, activation_id,
first_clause_offset, clause_count, source_ir_hash,
cnf_hash, encode_duration, variable_delta, aux_delta
```

Không nhất thiết clause cycle phải tham chiếu clause base bằng pointer. Cả hai cùng tham chiếu global variable IDs và được conjunction trong solver database.

### Lifecycle khuyến nghị

1. `open`: sidecar tạo session/factory/encoder/symbol table.
2. `encode_base`: validate, build formula, encode PG một lần, append ClauseStore, add incremental solver.
3. `add_cycle`: sidecar tự cấp activation, build implication, encode một lần, append/add.
4. `solve_incremental`: ordered complete activation assumptions, soft timeout, result/model projection.
5. `solve_fresh`: tạo MiniSat mới, replay selected canonical clauses, cùng assumptions.
6. `solve_z3`: replay đúng canonical integers/clauses qua Z3 Bool mapping.
7. `stats`: trả encoding counts tách solver metrics.
8. `close`: giải phóng toàn session.
9. Crash/hard timeout: Python restart và replay log; không cố khôi phục `SolverState` qua process.

## 14. Contract test bắt buộc trước rollout

### 14.1. Semantics/PG

Với công thức nhỏ, enumerate mọi assignment source `X`; kiểm:

```text
eval(F, X) == SAT(CNF(F) under X)
```

Bao phủ `true`, `false`, literal, duplicate, tautology, contradiction, nested NOT, implication, equivalence, AND/OR sâu, shared subformula và activation guard.

### 14.2. Base/cycle

* Một symbol xuất hiện ở base/cycle phải có cùng ID.
* `base ∧ (¬act_i ∨ cycle_i)` với `act_i=false` phải tương đương base.
* Với `act_i=true` phải tương đương `base ∧ cycle_i` trên source projection.
* Nhiều cycle: test current-only, cumulative và arbitrary subset.
* Encode theo các batch khác nhau nhưng cùng canonical input phải có cùng semantic result; nếu yêu cầu byte determinism, canonicalize rồi assert exact hash trong môi trường đã pin.

### 14.3. Cross solver

Cho mỗi corpus item:

* LogicNG fresh result == LogicNG incremental result == Z3 result.
* Nếu SAT, từng model projection phải thỏa Boolean IR gốc (không yêu cầu hai solver chọn cùng model).
* Nếu UNSAT, xác nhận bằng ít nhất backend thứ hai ở test corpus.
* `UNDEF/TIMEOUT` không được so như UNSAT.

### 14.4. Counts/provenance

* `variables_total = |SOURCE ∪ ACTIVATION ∪ ENCODER_AUX|` trên referenced IDs.
* Các tập đôi một rời nhau.
* Mọi literal ID trong clause tồn tại trong symbol table.
* Empty clause được giữ; true có zero clause.
* DIMACS debug round-trip giữ SAT result và mapping.

### 14.5. Incremental/timeout/recovery

* Assumption không trở thành permanent unit clause.
* Add cycle sau solve vẫn đúng.
* Save/load chỉ trên cùng MiniSat instance; state invalidation được test.
* Soft timeout trả `UNDEF`; query tiếp theo hoặc rebuild policy được test.
* Hard-kill sidecar rồi replay tạo kết quả giống fresh.
* Protocol malformed/oversized không làm lệch framing của request kế tiếp.

## 15. Quyết định triển khai cuối cùng

| Hạng mục | Quyết định |
|---|---|
| Bridge | Persistent JVM subprocess/sidecar; không JNI ở giai đoạn đầu |
| Input | Versioned Boolean IR từ parser Python hiện tại; không gửi Python/Verilog raw |
| Factory | Một named `FormulaFactory`/session, `PANIC`, single-thread ownership |
| Guard | `activation_i => cycle_i`, activation do sidecar cấp |
| Encoder | Explicit `CNFEncoder`, `PLAISTED_GREENBAUM`, `atomBoundary(0)` |
| Ground truth | Immutable canonical clause stream + global symbol table |
| LogicNG fresh | New MiniSat, replay exact selected clauses |
| LogicNG incremental | One append-only MiniSat/session + complete ordered assumptions |
| Z3 | Nạp exact same clause vectors và IDs; không tự encode lại Boolean IR |
| Statistics | Tính từ ClauseStore; source/activation/aux bằng registered set difference |
| Timeout | `RESTARTING_TIMEOUT` soft + Python process watchdog hard |
| Recovery | Restart JVM và replay canonical log; không serialize `SolverState` |
| Correctness oracle | Cross-solver SAT status + evaluate projected model against original IR |

Với các quyết định này, yêu cầu dùng Plaisted–Greenbaum, fresh SAT solver, incremental SAT solver và Z3 trên cùng CNF đều khả thi. Điểm cần tránh nhất là gọi `solver.add(originalFormula)` với `PG_ON_SOLVER` ở nhánh LogicNG trong khi Z3 nhận một CNF khác: cách đó đo hai encoding khác nhau và không còn đáp ứng mục tiêu “exact CNF mapping + ordered clause stream”.
