# Xác nhận bổ sung cho tích hợp LogicNG 2.6.2

## 1. Phạm vi và kết luận quyết định

Tài liệu này trả lời các vấn đề mới trong ảnh, dựa trực tiếp trên implementation LogicNG 2.6.2 tại commit `93bc457bb6dc29e658a6f8c1b836cec6ed2a4b62`. Số mục giữ theo ảnh (`9.1`, `9.3`–`9.8`; ảnh không có mục `9.2`).

Các kết luận cần chốt trước khi triển khai:

1. **CNF PG trả về bởi mỗi lần `encode(fragment)` là self-contained đối với mọi subformula reachable từ fragment đó.** Cache có thể tái dùng cùng auxiliary name và object `Formula`, nhưng khi cache hit, implementation trả lại chính clause định nghĩa đã cache; nó không chỉ trả auxiliary literal trần. Vì vậy `baseCnf + cycle2Cnf` không cần `cycle1Cnf` chỉ vì cycle 1 và 2 cùng chứa subformula `s`.
2. Vẫn phải kiểm thử contract self-contained trên version đã pin. Không đổi PG version/config hoặc gọi `clearCaches()` giữa chừng mà không chạy lại test.
3. Một activation chỉ tồn tại trong `FormulaFactory` **chưa** có nghĩa solver biết biến đó. Nếu assumption chứa tên chưa có trong solver, `sat(assumptions)` tạo một solver variable mới, không tạo clause; variable ấy tiếp tục nằm trong solver sau call.
4. Sau timeout trả `UNDEF`, public API không tuyên bố solver bị invalid. MiniSat 2.6.2 dọn decision level, handler và cancellation flag nên có thể gọi tiếp. Tuy nhiên learnt clauses/heuristic state được giữ; benchmark cần reproducibility nghiêm ngặt nên restore snapshot hoặc rebuild, còn production có thể reuse sau khi có regression test.
5. Public facade không có `addRawClause(int[])`. Cách supported là reconstruct một LogicNG clause rồi gọi `solver.add(clause)`. FormulaFactory và MiniSat core đều normalize; do đó chỉ cam kết **same logical input CNF**, không cam kết same physical internal database/order.
6. Không có random seed/randomization option trong `MiniSatConfig` 2.6.2. Muốn benchmark có thể so sánh, pin toàn bộ option ảnh hưởng search và môi trường; không tuyên bố bit-for-bit search trace.
7. `model(sourceVariables)` bỏ qua variable không có trong solver; không throw và không tự gán false. Adapter phải trả `UNASSIGNED/DONT_CARE`, hoặc completion riêng được đánh dấu.
8. Giữ hai representation/hash: physical ordered stream để replay chính xác, và canonical clause **multiset** để đối chiếu logic không phụ thuộc order. Không sort/deduplicate stream trước khi solve.

---

## 9.1. Dependency giữa các PG fragment

### 9.1.1. Câu hỏi chính

Với cùng `FormulaFactory` và `CNFEncoder`, nếu gọi:

```java
Formula baseCnf   = encoder.encode(base);
Formula cycle1Cnf = encoder.encode(f.implication(act1, cycle1));
Formula cycle2Cnf = encoder.encode(f.implication(act2, cycle2));
```

và `cycle1`, `cycle2` cùng chứa subformula `s`, liệu `cycle2Cnf` có chỉ tham chiếu auxiliary của `s` mà bỏ definition vì definition từng được sinh trong `cycle1Cnf` không?

### 9.1.2. Xác nhận từ implementation

**Không. Với public `PlaistedGreenbaumTransformation` của LogicNG 2.6.2, mỗi kết quả CNF vẫn chứa các PG definitions cần cho toàn bộ cây con reachable của input fragment.**

Luồng thực tế:

1. `CNFEncoder.encode()` giữ/reuse một `PlaistedGreenbaumTransformation` và gọi `formula.transform(...)`.
2. PG chuyển input sang NNF. Nếu chưa là CNF và vượt cutoff, `computeTransformation(root)` được gọi.
3. Với mỗi node `AND`/`OR`, `computeTransformation` luôn:
   * thêm `computePosPolarity(node)`; và
   * đệ quy `computeTransformation(op)` cho mọi operand.
4. Nếu definition của node đã cache, `computePosPolarity` trả về **`Formula result` chứa clause definition đã cache**, không trả riêng PG auxiliary variable.
5. Kết quả của fragment là conjunction của những definition đó; sau cùng top-level PG variable được restrict thành true.

Do đó, nếu `s` là shared subformula:

```text
encode(cycle1) -> ... AND Def(aux_s, s) AND ...
encode(cycle2) -> ... AND Def(aux_s, s) AND ...
```

Hai fragment có thể dùng cùng `aux_s`, nhưng cả hai đều mang lại `Def(aux_s, s)`. Cache tránh **tính lại/tạo lại object**, chứ không biến output lần sau thành một delta phụ thuộc output lần trước.

### 9.1.3. Ví dụ cụ thể

Giả sử:

```text
s      = p & q
cycle1 = s | r
cycle2 = s | t
g1     = act1 => cycle1
g2     = act2 => cycle2
```

Sau PG, ký hiệu auxiliary cho `s` là `v_s`. Dạng clause chính xác có thể khác do NNF/polarity/flattening, nhưng invariant cần có là CNF của `g2` chứa các clause đủ để ràng buộc `v_s` theo chiều polarity mà `g2` cần. Fresh solver nạp:

```text
baseCnf AND cycle2Cnf
```

và assumption `act2=true` không cần nạp `cycle1Cnf`.

### 9.1.4. Các ngoại lệ cần phân biệt

Kết luận trên áp dụng cho **materialized factory PG**:

```java
new CNFEncoder(f, CNFConfig.builder()
        .algorithm(CNFConfig.Algorithm.PLAISTED_GREENBAUM)
        .atomBoundary(0)
        .build())
```

Không suy rộng kết luận sang:

* `PG_ON_SOLVER`/`FULL_PG_ON_SOLVER`: đây là incremental transformation trực tiếp vào một solver, output không phải standalone `Formula` fragment để replay nơi khác;
* một encoder/fork khác có API trả delta clauses;
* major version LogicNG khác;
* việc tự đọc transformation cache hoặc chỉ serialize auxiliary literal mà bỏ `Formula` CNF trả về.

Nếu input đã là CNF, encoder trả input CNF. Nếu số atom nhỏ hơn `atomBoundary`, factorization được dùng; đặt `atomBoundary(0)` để tránh nhánh đó cho công thức không-CNF.

### 9.1.5. Có cần một CNFEncoder riêng cho từng fragment?

**Không cần cho tính self-contained, và phương án này không thực sự cô lập auxiliary nếu vẫn dùng cùng FormulaFactory.** PG variable/definition cache nằm trên formula/factory cache; hai encoder mới nhưng cùng factory vẫn có thể gặp cache đã tồn tại và tái dùng auxiliary.

Các lựa chọn:

| Cách | Source identity | Auxiliary reuse | Fragment self-contained | Khuyến nghị |
|---|---:|---:|---:|---|
| Một factory + một encoder/session | Có | Có | Có trong 2.6.2 | **Tốt nhất** |
| Một factory + encoder mới/fragment | Có | Vẫn có thể có | Có | Không đem thêm an toàn đáng kể |
| Factory riêng/fragment, factory name riêng | Phải remap theo source name | Không | Có | Chỉ dùng nếu muốn isolation tuyệt đối, tốn bộ nhớ/CPU |

Nếu dùng factory riêng cho fragment, phải đặt factory name duy nhất để auxiliary prefix không collision và phải map source variables theo tên vào global integer table. Không được cộng trực tiếp CNF từ các factory với auxiliary tên trùng.

### 9.1.6. Guard chống regression bắt buộc

Cho mỗi fragment `F_i`, test fragment hoàn toàn độc lập:

```text
for every source assignment X:
    eval(F_i, X) == SAT(CNF_i under X)
```

Và test tình huống shared subformula:

```text
SAT(baseCnf + cycle2Cnf, act2)
== SAT(base AND cycle2, act2)
```

Chạy test theo cả thứ tự `cycle1 -> cycle2` và `cycle2 -> cycle1`; chạy lại sau khi đổi JDK/LogicNG/config. Ngoài semantic test, có thể kiểm `variables(cycle2Cnf)` và clause definitions, nhưng semantic projection mới là oracle chính.

### 9.1.7. Quyết định triển khai

Giữ **một named FormulaFactory và một explicit PG CNFEncoder cho session**. Lưu nguyên CNF trả về theo batch. Fresh solver replay `base + selected guarded cycles`; không replay “mọi fragment trước selected cycle” chỉ để lấy auxiliary definition.

---

## 9.3. Assumption với unknown variable

### 9.3.1. “Đã đăng ký trong FormulaFactory” không đủ

FormulaFactory và SAT solver có hai symbol table khác nhau. Việc gọi:

```java
Variable act7 = f.variable("act7");
```

chỉ intern variable trong factory. Solver chỉ biết tên sau khi tên xuất hiện trong clause đã add, hoặc khi một API solver gọi `getOrAddIndex` cho literal đó.

### 9.3.2. Side effect chính xác

`MiniSat.sat(handler, assumptions)` chuyển assumption collection thành internal vector. Với từng literal, `getOrAddIndex`:

1. tìm name trong solver `name2idx`;
2. nếu không có, gọi core `newVar(...)`;
3. thêm mapping name → solver index;
4. assumption được áp cho solve hiện tại;
5. sau solve, internal assumptions vector được clear, **nhưng variable và name mapping mới vẫn còn**.

Không có clause permanent nào được tạo chỉ vì assumption. Tuy nhiên biến mới xuất hiện trong `knownVariables()`, có thể ảnh hưởng model enumeration, model shape, variable count và search order về sau. Đây chính là side effect public Javadoc cảnh báo.

### 9.3.3. Activation đã nằm trong guarded CNF

Nếu `cycle7Cnf` thật sự chứa literal `act7` và đã được `solver.add(cycle7Cnf)` trước query, solver đã biết `act7`; assumption không tạo variable mới. Chỉ “có object `act7` trong factory” thì chưa đủ.

Do đó policy trong ảnh là đúng:

* chỉ assume activation của guarded fragment đã add;
* không đưa future activation vào growing assumption vector;
* nếu dùng final database, add tất cả guarded CNF trước query đầu tiên;
* assert `solver.knownVariables().contains(act)` trước solve trong debug/contract tests.

Không nên add tautological registration clause `(act | ~act)`: FormulaFactory rút nó thành true và solver vẫn không biết `act`. Cũng không nên add một unit “dummy” vì nó làm thay đổi semantics. Cách đúng là activation phải xuất hiện thật trong guard, hoặc chấp nhận registration bằng assumption rồi quản lý side effect rõ ràng.

### 9.3.4. Ví dụ policy

```java
inc.add(cycle7Cnf);             // cycle7Cnf là CNF(act7 => cycle7)
if (!inc.knownVariables().contains(act7)) {
    throw new IllegalStateException("activation was simplified out");
}
Tristate result = inc.sat(Arrays.asList(act7));
```

Activation có thể bị simplify khỏi guard nếu `cycle7` là hằng true, vì `act7 => true` là true. Khi đó không cần activation để bật constraint (constraint không ràng buộc gì). Adapter nên đánh dấu batch `NO_OP` và không assume activation đó, thay vì tạo dummy clause.

---

## 9.4. Trạng thái solver sau `UNDEF`

### 9.4.1. Public contract và implementation 2.6.2

Public API định nghĩa `UNDEF` là computation bị SATHandler cancel; không có Javadoc nào nói solver trở thành unusable hoặc cấm `add()`/`sat()` tiếp theo.

MiniSat 2.6.2 khi kết thúc `solve()`—kể cả canceled—thực hiện:

* gọi `finishSolving(handler)`;
* `cancelUntil(0)` để quay về decision level 0;
* bỏ reference handler;
* đặt `canceledByHandler = false`;
* assumption wrapper clear assumption vector.

Vì vậy **reuse cùng solver sau `UNDEF` là được implementation hỗ trợ trên thực tế**: có thể gọi `sat()` lại hoặc `add()` rồi solve. `MiniSat.add()` cũng đặt cached result về `UNDEF` trước khi thêm.

### 9.4.2. Điều không được rollback

Timeout không khôi phục solver về trạng thái “chưa từng search”:

* learnt clauses hợp lệ có thể còn;
* activities, phase/heuristic state và counters có thể đã đổi;
* thời gian query tiếp theo vì vậy khác fresh solver;
* model cũ bị clear ở đầu solve; sau `UNDEF`, không được gọi `model()` để lấy partial model—facade sẽ từ chối vì result là `UNDEF`.

### 9.4.3. Policy phù hợp theo mục tiêu

| Mục tiêu | Policy |
|---|---|
| Production throughput, chấp nhận learnt state | Có thể reuse sau `UNDEF`, nhưng phải có regression/stress test |
| Benchmark cần so execution độc lập | Save state trước solve rồi load lại, hoặc discard/replay |
| Hard watchdog đã kill JVM | Bắt buộc tạo process/solver mới và replay |
| Không tin custom handler/thread interruption | Discard/replay là an toàn nhất |

Nếu dùng `saveState/loadState`, chỉ MiniSat/MiniCard incremental hỗ trợ và snapshot thuộc đúng instance. Snapshot trước solve rồi load sau `UNDEF` loại các cấu trúc tăng sau snapshot theo cơ chế state hiện tại, nhưng không phải serialized checkpoint và cần benchmark/test riêng.

### 9.4.4. Kết luận sửa so với policy quá bảo thủ

Không cần tuyên bố bắt buộc `UNDEF => discard` cho mọi trường hợp. Quy tắc chính xác hơn:

```text
UNDEF => không có SAT/UNSAT/model
      => solver 2.6.2 có thể reuse
      => restore/rebuild nếu cần isolation hoặc nếu cancellation không nằm trong flow đã test
```

---

## 9.5. Exact clause order khi nạp vào LogicNG

### 9.5.1. Public API supported

`SATSolver.addClause(...)` không public; `MiniSat.generateClauseVector(...)` và core API không nên trở thành integration contract. Cách public tốt nhất là dựng một clause `Formula` và gọi `solver.add(clause)` theo đúng thứ tự stream:

```java
for (int[] rawClause : orderedClauseStream) {
    Formula clause;
    if (rawClause.length == 0) {
        clause = f.falsum();
    } else if (rawClause.length == 1) {
        clause = literalFor(rawClause[0]);
    } else {
        List<Literal> literals = literalsFor(rawClause);
        clause = f.clause(literals); // input đã được validate/normalize
    }
    solver.add(clause);
}
```

Solver config đặt `FACTORY_CNF`; mỗi formula trên đã là CNF nên không PG encode lần nữa.

### 9.5.2. Những normalization sẽ xảy ra

Không thể cam kết physical internal database giống input:

* FormulaFactory dùng set semantics, loại duplicate literals và simplify complementary pair/constant.
* MiniSat core **sort literals trong mỗi clause**.
* Core loại duplicate literal, bỏ false-at-level-0 literal, bỏ cả clause nếu đã true hoặc tautological.
* Empty clause đặt solver `ok=false`.
* Unit clause được enqueue/propagate và lưu riêng trong incremental mode, không nằm như một ordinary `MSClause`.
* Clause thêm sau unit propagation có thể được rút ngắn hoặc bỏ vì assignment level 0.
* Solver có thể thêm learnt clauses và remove satisfied clauses trong quá trình solve.

Thứ tự gọi `solver.add(...)` vẫn là thứ tự clause input, nhưng internal literal order và physical clause collection không phải byte-for-byte stream. Z3 cũng được quyền preprocess khác.

### 9.5.3. Cam kết phải dùng

Cam kết đúng là:

```text
same canonical input CNF + same variable mapping + same assumptions
```

không phải:

```text
same physical internal solver database
```

Log/hash stream **trước** khi đưa vào backend. So sánh SAT status và projected model với original IR. Không dùng `FormulaOnSolverFunction` hoặc `underlyingSolver().clauses()` để chứng minh backend đã giữ physical stream nguyên dạng.

### 9.5.4. Validate trước khi reconstruct

Canonical encoder output bình thường đã không chứa duplicate/tautological literal. Dù vậy adapter phải validate:

* mọi ID tồn tại;
* không có `0` trong clause;
* không duplicate literal;
* không có cả `x` và `-x`;
* empty clause được giữ;
* không mutate ordered stream khi dựng LogicNG objects.

Nếu cần API raw clause tuyệt đối không normalize, LogicNG 2.6.2 public facade không đáp ứng. Gọi `underlyingSolver().addClause(...)` là technically possible nhưng bị chính API cảnh báo có thể làm hỏng invariants; không khuyến nghị và không coi là supported bridge.

---

## 9.6. Cấu hình MiniSat đầy đủ và reproducibility

### 9.6.1. Các option ảnh hưởng search

Ngoài `incremental(true)` và `cnfMethod(FACTORY_CNF)`, `MiniSatConfig` có:

| Option | Default thực tế trong builder | Ảnh hưởng |
|---|---:|---|
| `varDecay` | `0.95` | Variable activity decay |
| `varInc` | `1.0` | Initial variable activity increment |
| `clMinimization` | `DEEP` | Learnt clause minimization |
| `restartFirst` | `100` | Base restart interval |
| `restartInc` | `2.0` | Luby/restart scale |
| `clauseDecay` | `0.999` | Clause activity decay |
| `removeSatisfied` | `true` | Loại original clause đã satisfied ở simplification level 0 |
| `lsFactor` | `1/3` | Initial learnt limit factor |
| `lsInc` | `1.1` | Learnt limit growth |
| `incremental` | `true` | Incremental behavior/save-load |
| `initialPhase` | `false` | Initial variable phase trong code builder |
| `proofGeneration` | `false` | DRUP/proof bookkeeping |
| `cnfMethod` | `PG_ON_SOLVER` | Phải override thành `FACTORY_CNF` |
| `auxiliaryVariablesInModels` | `false` | Model filtering theo reserved prefixes |

Lưu ý Javadoc cạnh setter `initialPhase` nói default true nhưng field builder thực tế khởi tạo `false`; với version này phải tin code chạy và pin explicit value để tránh ambiguity.

### 9.6.2. Có randomization/seed không?

`MiniSatConfig` 2.6.2 không có random seed, random branching frequency hay option randomization. Core chọn biến theo activity/heap, initial phase và optional selection order. Vì thế:

* không cần pin seed vì không có seed public để pin;
* vẫn không có guarantee bit-for-bit giữa JVM/version/history do hash iteration, learnt state, order add, timing/cancel và implementation changes;
* không dùng `setSelectionOrder` trừ khi project muốn một branching prefix cụ thể; nếu dùng, luôn reset và log order.

### 9.6.3. Cấu hình benchmark đề xuất

Pin tường minh thay vì dựa vào default:

```java
MiniSatConfig cfg = MiniSatConfig.builder()
        .varDecay(0.95)
        .varInc(1.0)
        .clMinimization(MiniSatConfig.ClauseMinimization.DEEP)
        .restartFirst(100)
        .restartInc(2.0)
        .clauseDecay(0.999)
        .removeSatisfied(true)
        .lsFactor(1.0 / 3.0)
        .lsInc(1.1)
        .incremental(true)
        .initialPhase(false)
        .proofGeneration(false)
        .cnfMethod(MiniSatConfig.CNFMethod.FACTORY_CNF)
        .auxiliaryVariablesInModels(false)
        .build();
```

Không đổi `removeSatisfied` chỉ để mong physical database giống Z3; Z3 vẫn có preprocessing riêng. Nếu mục tiêu nghiên cứu tác động option, mỗi config là một benchmark dimension riêng và phải ghi full config vào result manifest.

### 9.6.4. Các biến môi trường cũng phải pin

Pin LogicNG artifact/commit, JDK vendor/version, heap/GC flags, CPU affinity, warm-up, clause stream hash, assumptions order, solve sequence và timeout mode. Incremental solver history là một phần input benchmark; hai run chỉ so được khi có cùng lịch sử add/solve/timeout.

---

## 9.7. Model khi source variable bị simplify khỏi CNF

### 9.7.1. Behavior chính xác

`solver.model(sourceVariables)` chuyển từng source variable thành solver index. Nếu solver không biết tên, index là `-1`; method **bỏ qua variable đó**, không throw và không thêm literal false/true. Nếu solver biết variable, SAT model core có Boolean assignment và adapter nhận literal tương ứng (trừ khi bị model relevance filter loại theo reserved prefix).

Vì thế `Assignment` trả về là partial mapping đối với requested source set:

```text
present positive literal -> TRUE
present negative literal -> FALSE
absent requested name    -> UNASSIGNED / DONT_CARE
```

Không được diễn giải absent là false.

### 9.7.2. Vì sao completion là hợp lệ

Nếu một source variable biến mất do simplification semantics-preserving, satisfiability của công thức đã simplify không phụ thuộc biến đó. Có thể complete bằng false hoặc true. Nhưng completion là quyết định của adapter, không phải assignment được solver chứng minh/chọn.

Response nên biểu diễn rõ:

```json
{
  "model": {"x":"TRUE","y":"FALSE","z":"DONT_CARE"},
  "completion_policy":"NONE"
}
```

Nếu downstream bắt buộc Boolean total model:

```json
{
  "model": {"x":true,"y":false,"z":false},
  "solver_assigned":["x","y"],
  "completed":{"z":false},
  "completion_policy":"FALSE_FOR_DONT_CARE"
}
```

### 9.7.3. Validation với original IR

Để validate original formula:

1. Lấy partial source model.
2. Với dont-care, chọn completion deterministic (ví dụ false), có ghi metadata.
3. Evaluate original Boolean IR với total assignment.
4. Nếu evaluation false, thử completion/SAT extension đúng cách; không được lập tức kết luận solver sai nếu adapter đã chọn completion tùy ý cho biến mà simplification/projection xử lý đặc biệt.

Phép kiểm mạnh nhất là hỏi SAT với assignments của các biến đã có và tìm extension cho biến còn thiếu, hoặc với corpus nhỏ enumerate completions. Activation và auxiliary không nằm trong source model.

Source namespace phải cấm `@RESERVED_*`; nếu không, `isRelevantVariable` có thể lọc nhầm source variable khỏi model khi `auxiliaryVariablesInModels(false)`.

---

## 9.8. Canonicalization policy

### 9.8.1. Hai representation là đúng

Giữ đồng thời:

1. `ordered_clause_stream`: physical encoder output sau extraction và global-ID mapping; đây là payload replay vào LogicNG/Z3.
2. `canonical_clause_multiset`: bản chuẩn hóa chỉ để compare/hash/diagnostic; không dùng thay stream khi solve.

Hai hash:

```text
ordered_clause_stream_hash
canonical_clause_multiset_hash
```

Hash luôn kèm manifest `schema_version`, LogicNG version, encoding config và symbol-table version để tránh cùng byte nhưng khác semantics protocol.

### 9.8.2. Variable ID policy

* **Source:** sort tên theo lexicographic order của UTF-8 bytes (không locale/collator), rồi cấp ID tăng dần.
* **Activation:** cấp theo numeric cycle ID; nếu cycle ID không dense, sort `(cycle_id, activation_name_utf8)`.
* **Auxiliary:** deterministic first occurrence khi duyệt `ordered_clause_stream`; không dựa vào số suffix LogicNG.
* Các nhóm dùng range hoặc field `kind`, nhưng một global ID duy nhất. Không tái sử dụng ID.

Nếu source/activation được biết dần theo streaming, lexicographic global allocation không thể giữ mà không renumber. Chọn một trong hai contract:

* khai báo toàn bộ source symbol trước `encode_base` rồi sort; activation range cấp riêng theo cycle; hoặc
* stable first-seen allocation và chấp nhận ID không lexicographic.

Không renumber clauses đã add vào incremental solver.

### 9.8.3. Ordered stream

Đối với mỗi clause lấy từ LogicNG CNF:

* giữ nguyên thứ tự traversal của formula;
* giữ thứ tự literal extraction;
* giữ thứ tự các batch (`base`, rồi cycle add order);
* không sort, không deduplicate, không bỏ tautology trong bước này;
* serialize length-prefix để empty clause phân biệt được với end-of-stream.

Đây là artifact để phát hiện encoder output/order thay đổi. Solver core có thể normalize sau khi nhận.

### 9.8.4. Canonical clause multiset

Tạo copy, không mutate ordered stream:

1. Trong mỗi clause, sort literal theo key `(abs(id), signRank)`, quy ước `negative` trước `positive` (hoặc ngược lại nhưng phải versioned).
2. Giữ empty clause.
3. Sort các clause lexicographically theo `(length, literal sequence)` hoặc chỉ literal sequence; chọn một quy tắc versioned.
4. **Giữ multiplicity của duplicate clauses** để đây là multiset hash. Không deduplicate.
5. Nếu muốn hash semantic set bỏ duplicate/tautology, tạo hash thứ ba có tên rõ `normalized_clause_set_hash`; không dùng nó thay multiset hash.

Ví dụ:

```text
ordered:
  [ 5, -2, 3 ]
  [ -7 ]
  [ 3, 5, -2 ]

canonical multiset (negative-before-positive, abs-id):
  [ -2, 3, 5 ]
  [ -2, 3, 5 ]
  [ -7 ]
```

Hai clause đầu vẫn xuất hiện hai lần.

### 9.8.5. Encoding bytes cho hash

Không hash JSON pretty-print. Dùng binary canonical encoding, ví dụ:

```text
magic/version
symbol_count
for symbol: kind, id, UTF-8-byte-length, UTF-8 bytes
clause_count
for clause: literal_count, signed integers in fixed endian/varint contract
```

Dùng SHA-256. Hash assumptions riêng theo ordered literal vector vì assumption order có thể ảnh hưởng search dù không ảnh hưởng logic.

---

## 10. Checklist triển khai và acceptance tests

### 10.1. Encoder fragment

* [ ] LogicNG/Maven artifact pin `2.6.2` và commit/source reference được ghi manifest.
* [ ] `PLAISTED_GREENBAUM`, `atomBoundary(0)`, cùng named FormulaFactory/session.
* [ ] Mỗi fragment được solve standalone với mọi shared-subformula order trong test corpus.
* [ ] Không serialize transformation cache; chỉ serialize CNF fragment đã trả về.
* [ ] Không dùng `PG_ON_SOLVER` để tạo artifact dùng chung với Z3.

### 10.2. Assumptions

* [ ] Activation phải có trong guarded CNF đã add trước assumption.
* [ ] Activation của no-op guard không được assume; batch mang trạng thái `NO_OP`.
* [ ] Không thêm future activations vào solver sớm ngoài policy đã chọn.
* [ ] Assumption collection có order ổn định và có hash.
* [ ] Test `knownVariables()` trước/sau unknown assumption để phát hiện side effect.

### 10.3. Timeout

* [ ] `UNDEF` không map thành UNSAT, không đọc model.
* [ ] Reuse-after-UNDEF stress test: add, solve SAT/UNSAT và compare fresh solver.
* [ ] Benchmark isolation dùng save/load hoặc replay; policy được ghi trong result.
* [ ] Hard-killed JVM luôn replay trên process mới.

### 10.4. Clause loading/model

* [ ] Log/hash trước backend normalization.
* [ ] Empty clause được reconstruct thành `f.falsum()`.
* [ ] Không kiểm same internal database giữa LogicNG và Z3.
* [ ] Missing source model entry là `DONT_CARE`, không phải false.
* [ ] Completion policy có metadata và original IR được evaluate/extension-check.

### 10.5. Determinism

* [ ] Full MiniSat config được serialize vào manifest.
* [ ] JDK/JVM/CPU/warm-up/session history được pin hoặc ghi nhận.
* [ ] Ordered stream và canonical multiset có hash riêng.
* [ ] Canonicalization không mutate payload solve.
* [ ] Duplicate clause multiplicity không bị mất trong multiset hash.

## 11. Quyết định cuối cùng

Thiết kế “encode từng part một lần, fresh replay base + selected cycle” **an toàn với LogicNG 2.6.2** khi dùng explicit `PlaistedGreenbaumTransformation` qua `CNFEncoder`: mỗi returned CNF fragment là self-contained dù auxiliary/cache được reuse. Đây là điểm quan trọng nhất của đợt xác nhận này.

Không cần tạo encoder riêng cho từng fragment. Điều bắt buộc là giữ cùng source/global-ID contract, materialize và lưu từng full CNF fragment, test standalone projection, và không nhầm solver-internal PG delta với factory PG output.

Đối với các vấn đề còn lại, LogicNG đáp ứng được incremental assumptions, continuation sau soft timeout và model projection, nhưng adapter phải làm rõ side effects/partial model. Exact physical database và bit-for-bit search reproducibility không thể cam kết qua public API; thay vào đó dùng exact pre-backend stream, hai loại hash, full configuration manifest và semantic cross-solver tests.
