# 📚 Library Management System — Detailed Learning Roadmap (Java OOP Advanced)

> **Mục tiêu**: Xây dựng hệ thống quản lý thư viện bằng Java thuần (chưa cần Spring Boot), luyện chắc **Generics**, **State Pattern**, **Service điều phối nhiều Repository**, **Custom Exception Handling** và **Observer Pattern** — làm nền tảng sát nhất có thể trước khi bước sang Spring Boot thật.

---

## 🏗️ Tổng quan Kiến trúc Hệ thống

```
LibraryManagementSystem/
├── src/
│   └── main/java/com/library/
│       ├── model/
│       │   ├── Entity.java              ← Interface gốc có ID
│       │   ├── Book.java                ← Book data model & methods
│       │   ├── Member.java              ← Member data model & methods
│       │   └── Loan.java                ← Loan (phiếu mượn) nối Book ↔ Member
│       ├── enums/
│       │   ├── BookStatus.java          ← AVAILABLE, BORROWED, RESERVED, LOST
│       │   └── LoanStatus.java          ← ACTIVE, RETURNED, OVERDUE
│       ├── state/
│       │   ├── BookState.java           ← Interface State Pattern cho Book
│       │   ├── AvailableState.java
│       │   ├── BorrowedState.java
│       │   └── ReservedState.java
│       ├── exception/
│       │   ├── BookNotAvailableException.java
│       │   ├── MemberLimitExceededException.java
│       │   ├── LoanNotFoundException.java
│       │   └── LoanAlreadyReturnedException.java
│       ├── repository/
│       │   ├── IRepository.java         ← Generic Interface IRepository<T>
│       │   └── InMemoryRepository.java  ← Generic Class Repository<T>
│       ├── service/
│       │   ├── MockApiService.java      ← Giả lập gọi API có độ trễ (Thread.sleep)
│       │   └── LibraryService.java      ← Business logic điều phối Book + Member + Loan
│       ├── event/
│       │   ├── LibraryEventListener.java ← Interface Observer
│       │   └── EventPublisher.java       ← Pub/Sub Event engine
│       └── Main.java                     ← Chạy thử toàn bộ flow mượn/trả sách
└── src/test/java/com/library/
    ├── InMemoryRepositoryTest.java
    ├── BookStateTest.java
    ├── LibraryServiceTest.java
    └── EventPublisherTest.java
```

> Dùng JUnit 5 cho toàn bộ test. Có thể build bằng Maven hoặc Gradle — nếu chưa quen build tool, dùng Maven (`pom.xml`) trước vì tài liệu tiếng Việt nhiều hơn.

---

## 📌 PHASE 1 — Generic Repository Pattern (`<T>`)

> **Mục tiêu**: Nắm vững **Generics trong Java (`<T extends X>`)** — viết một tầng lưu trữ dùng chung được cho mọi entity (Book, Member, Loan) mà không lặp code.

### 🎯 Các bước thực hiện:

1. **Tạo Interface `Entity`** (file `model/Entity.java`):
   ```java
   public interface Entity {
       String getId();
       LocalDateTime getCreateAt();
       LocalDateTime getUpdateAt();
   }
   ```

2. **Thiết kế Generic Interface `IRepository<T extends Entity>`** (file `repository/IRepository.java`):
   - `void add(T item)`
   - `Optional<T> findById(String id)` — dùng `Optional` thay vì trả `null` trực tiếp, đúng convention Java
   - `List<T> findAll()`
   - `boolean update(String id, Consumer<T> updater)` — truyền một hàm cập nhật (functional interface) thay vì `Partial<T>` như TypeScript, vì Java không có kiểu Partial
   - `boolean delete(String id)`
   - `List<T> filter(Predicate<T> predicate)`

3. **Cài đặt Class `InMemoryRepository<T extends Entity>`** (file `repository/InMemoryRepository.java`):
   - Dùng `private final Map<String, T> items = new HashMap<>();` (tra cứu theo ID nhanh hơn List)
   - Triển khai toàn bộ method từ `IRepository<T>`
   - `update()` nhận vào `Consumer<T> updater`, lấy object hiện có rồi gọi `updater.accept(item)` — cách này tránh được lỗi ghi đè nhầm `id`/`createAt` vì người gọi tự chọn field nào cần đổi ngay trên object, không phải merge object mới đè lên object cũ.

4. **Khởi tạo 3 repository độc lập trong `LibraryService`**:
   ```java
   private final IRepository<Book> bookRepo = new InMemoryRepository<>();
   private final IRepository<Member> memberRepo = new InMemoryRepository<>();
   private final IRepository<Loan> loanRepo = new InMemoryRepository<>();
   ```

---

## 📌 PHASE 2 — Core Library Domain Models

> **Mục tiêu**: Xây dựng mô hình dữ liệu thư viện có quan hệ thật sự giữa các entity (Book, Member, Loan tham chiếu chéo nhau qua ID).

### 🎯 Các bước thực hiện:

1. **Định nghĩa Enum** (package `enums`):
   ```java
   public enum BookStatus { AVAILABLE, BORROWED, RESERVED, LOST }
   public enum LoanStatus { ACTIVE, RETURNED, OVERDUE }
   ```

2. **Class `Book implements Entity`** (file `model/Book.java`):
   - Field: `id`, `title`, `author`, `isbn`, `status`, `createAt`, `updateAt` — tất cả `private final` trừ `status` và `updateAt`
   - **Không** viết setter public cho `status` — chỉ cho phép đổi trạng thái thông qua package-private method `setStatus(BookStatus)` mà State Pattern (Phase 3) gọi tới, để tránh code bên ngoài đổi trạng thái tùy tiện.
   - Method `getInfo(): String` — trả về chuỗi mô tả ngắn gọn (title + author + status) để log/debug.

3. **Class `Member implements Entity`** (file `model/Member.java`):
   - Field: `id`, `name`, `email`, `private final List<String> activeLoanIds = new ArrayList<>();`, `createAt`, `updateAt`
   - Method `boolean canBorrow(int maxLimit)` → `return activeLoanIds.size() < maxLimit;`
   - Method `void addActiveLoan(String loanId)`
   - Method `void removeActiveLoan(String loanId)`

4. **Class `Loan implements Entity`** (file `model/Loan.java`):
   - Field: `id`, `bookId`, `memberId`, `borrowDate: LocalDate`, `dueDate: LocalDate`, `returnDate: LocalDate` (nullable), `status: LoanStatus`, `createAt`, `updateAt`
   - Method `boolean isOverdue(LocalDate referenceDate)` → `status == LoanStatus.ACTIVE && dueDate.isBefore(referenceDate)`
   - Method `long calculateFine(long finePerDay)`:
     - Nếu `returnDate == null` → tính số ngày trễ từ `dueDate` đến hôm nay bằng `ChronoUnit.DAYS.between(dueDate, LocalDate.now())`
     - Nếu đã trả → tính từ `dueDate` đến `returnDate`
     - Nếu không trễ → trả về `0`
     - Công thức: `Math.max(0, soNgayTre) * finePerDay`

---

## 📌 PHASE 3 — State Pattern cho Book

> **Mục tiêu**: Thay vì cho phép đổi `status` sang bất kỳ giá trị nào tùy tiện, mỗi trạng thái của Book phải tự quyết được phép chuyển tiếp sang trạng thái nào, và tự throw exception khi hành động không hợp lệ.

### 🎯 Các bước thực hiện:

1. **Định nghĩa Interface `BookState`** (file `state/BookState.java`):
   ```java
   public interface BookState {
       void borrow(Book book, String requestingMemberId);
       void returnBook(Book book);
       void reserve(Book book);
   }
   ```
   - Có tham số `requestingMemberId` ngay trong interface (khác với thiết kế TypeScript) vì `ReservedState` cần biết ai đang yêu cầu mượn để so sánh với người đã đặt trước.

2. **Cài đặt `AvailableState implements BookState`** (file `state/AvailableState.java`):
   - `borrow()` → hợp lệ: `book.setStatus(BookStatus.BORROWED)`
   - `reserve()` → hợp lệ: `book.setStatus(BookStatus.RESERVED)`
   - `returnBook()` → không hợp lệ: `throw new IllegalStateException("Sách chưa được mượn nên không thể trả")`

3. **Cài đặt `BorrowedState implements BookState`** (file `state/BorrowedState.java`):
   - `borrow()` → không hợp lệ: `throw new BookNotAvailableException(book.getId())`
   - `returnBook()` → hợp lệ: `book.setStatus(BookStatus.AVAILABLE)`
   - `reserve()` → không hợp lệ: `throw new IllegalStateException("Không thể đặt trước sách đang được mượn")`

4. **Cài đặt `ReservedState implements BookState`** (file `state/ReservedState.java`):
   - Cần biết `reservedByMemberId` — thêm field này vào `Book` (nullable), set khi `reserve()` được gọi ở `AvailableState`
   - `borrow(book, requestingMemberId)` → chỉ hợp lệ nếu `requestingMemberId.equals(book.getReservedByMemberId())`; ngược lại `throw new BookNotAvailableException(book.getId())`
   - `returnBook()` → không hợp lệ (sách đang đặt trước, chưa ai mượn)

5. **`BookStateFactory`** (có thể là static method trong `BookState` hoặc class riêng):
   ```java
   static BookState from(BookStatus status) {
       return switch (status) {
           case AVAILABLE -> new AvailableState();
           case BORROWED -> new BorrowedState();
           case RESERVED -> new ReservedState();
           default -> throw new IllegalStateException("Trạng thái không hỗ trợ: " + status);
       };
   }
   ```
   - Dùng `switch` expression (Java 14+) cho gọn — nếu môi trường học dùng Java cũ hơn, thay bằng `switch` statement truyền thống hoặc `if-else`.

---

## 📌 PHASE 4 — Custom Exception Handling

> **Mục tiêu**: Java có sẵn cơ chế Exception mạnh hơn TypeScript — cần luyện đúng cách dùng **Checked vs Unchecked Exception**, vì đây là kiến thức sẽ dùng trực tiếp khi viết `@ControllerAdvice` ở Spring Boot sau này.

### 🎯 Các bước thực hiện:

1. **Tạo class cha `LibraryException extends RuntimeException`** (package `exception`):
   - Dùng `RuntimeException` (unchecked) vì đây là lỗi nghiệp vụ, không bắt buộc caller phải try-catch ở mọi nơi gọi — giống cách Spring Boot xử lý exception nghiệp vụ.

2. **Viết các class con**:
   - `BookNotAvailableException extends LibraryException` — constructor nhận `String bookId`, gọi `super("Sách " + bookId + " hiện không khả dụng để mượn")`
   - `MemberLimitExceededException extends LibraryException` — nhận `String memberId, int limit`
   - `LoanNotFoundException extends LibraryException` — nhận `String loanId`
   - `LoanAlreadyReturnedException extends LibraryException` — nhận `String loanId`

3. **Quy tắc dùng exception**: Mọi method trong `LibraryService` khi gặp điều kiện nghiệp vụ không thỏa phải `throw` đúng loại exception tương ứng — không được trả `false`/`null` để báo lỗi im lặng. Đây chính là thói quen cần có trước khi viết `@ExceptionHandler` trong Spring Boot.

---

## 📌 PHASE 5 — Service điều phối nhiều Repository

> **Mục tiêu**: Đây là phần lõi của dự án — một nghiệp vụ (mượn sách) phải đụng vào cả 3 entity (Book, Member, Loan) trong cùng một luồng xử lý, có validate và có publish event.

### 🎯 Các bước thực hiện:

1. **Viết `MockApiService`** (file `service/MockApiService.java`):
   - `void delay(long millis) throws InterruptedException` — gọi `Thread.sleep(millis)` để mô phỏng độ trễ mạng 500–1000ms
   - `boolean saveLoan(Loan loan) throws InterruptedException` — gọi `delay()` rồi trả `true`, mô phỏng lưu phiếu mượn xuống server

2. **Sinh ID đúng cách** — viết một class tiện ích `IdGenerator`:
   ```java
   public class IdGenerator {
       public static String generate(String prefix) {
           return prefix + "-" + System.currentTimeMillis() + "-" + 
                  ThreadLocalRandom.current().nextInt(10000);
       }
   }
   ```
   - Dùng cho mọi entity mới tạo (Book, Member, Loan) — không được gán ID cố định bằng chuỗi tĩnh.

3. **Viết `LibraryService.borrowBook(String bookId, String memberId): Loan`**:
   - B1: `Book book = bookRepo.findById(bookId).orElseThrow(() -> new BookNotAvailableException(bookId));`
   - B2: `BookState state = BookState.from(book.getStatus()); state.borrow(book, memberId);` — nếu không hợp lệ, State tự throw exception
   - B3: `Member member = memberRepo.findById(memberId).orElseThrow(...)`
   - B4: `if (!member.canBorrow(5)) throw new MemberLimitExceededException(memberId, 5);`
   - B5: Tạo `Loan` mới: `borrowDate = LocalDate.now()`, `dueDate = borrowDate.plusDays(14)`, `status = ACTIVE`, `id = IdGenerator.generate("LOAN")`
   - B6: `loanRepo.add(loan);` `bookRepo.update(bookId, b -> {});` (trạng thái đã đổi ngay trên object tham chiếu); `member.addActiveLoan(loan.getId());` `memberRepo.update(memberId, m -> {});`
   - B7: `apiService.saveLoan(loan);` — giả lập đồng bộ lên server (nhớ khai báo `throws InterruptedException` hoặc bọc try-catch)
   - B8: `eventPublisher.publish("BOOK_BORROWED", new BookBorrowedEvent(bookId, memberId, loan.getId()));`
   - B9: `return loan;`

4. **Viết `LibraryService.returnBook(String loanId): long`** (trả về số tiền phạt):
   - B1: `Loan loan = loanRepo.findById(loanId).orElseThrow(() -> new LoanNotFoundException(loanId));`
   - B2: `if (loan.getStatus() != LoanStatus.ACTIVE) throw new LoanAlreadyReturnedException(loanId);`
   - B3: `loan.setReturnDate(LocalDate.now()); loan.setStatus(LoanStatus.RETURNED);`
   - B4: `long fine = loan.calculateFine(5000);` (ví dụ 5.000đ/ngày trễ)
   - B5: Lấy `Book`, gọi `BookState.from(book.getStatus()).returnBook(book);` để đổi về `AVAILABLE`
   - B6: Cập nhật lại repository tương ứng, gọi `member.removeActiveLoan(loanId);`
   - B7: `eventPublisher.publish("BOOK_RETURNED", new BookReturnedEvent(loanId, fine));`
   - B8: `return fine;`

5. **Viết `LibraryService.getOverdueLoans(): List<Loan>`**:
   - `loanRepo.filter(loan -> loan.isOverdue(LocalDate.now()));`

6. **Viết `LibraryService.searchBooks(String keyword): List<Book>`**:
   - `bookRepo.filter(book -> book.getTitle().toLowerCase().contains(keyword.toLowerCase()) || book.getAuthor().toLowerCase().contains(keyword.toLowerCase()));`

---

## 📌 PHASE 6 — Observer Pattern (Event Publishing)

> **Mục tiêu**: Java có `java.util.Observer` nhưng đã bị deprecated — luyện tự viết Observer Pattern bằng tay theo cách hiện đại, đúng với cách các framework thật (Spring Events) hoạt động.

### 🎯 Các bước thực hiện:

1. **Định nghĩa Interface `LibraryEventListener<T>`** (file `event/LibraryEventListener.java`):
   ```java
   public interface LibraryEventListener<T> {
       void onEvent(T eventData);
   }
   ```

2. **Viết Class `EventPublisher`** (file `event/EventPublisher.java`):
   - `private final Map<String, List<LibraryEventListener<Object>>> listeners = new HashMap<>();`
   - `void subscribe(String eventName, LibraryEventListener<Object> listener)` — thêm vào danh sách theo key `eventName`
   - `void publish(String eventName, Object data)` — lấy danh sách listener theo `eventName`, lần lượt gọi `listener.onEvent(data)`
   - `void unsubscribe(String eventName, LibraryEventListener<Object> listener)`

3. **Tích hợp vào `LibraryService`**:
   - Publish `"BOOK_BORROWED"` khi mượn thành công (đã nêu ở Phase 5)
   - Publish `"BOOK_RETURNED"` khi trả thành công, kèm số tiền phạt
   - Publish `"LOAN_OVERDUE"` — gọi định kỳ (hoặc trong `Main.java` demo) khi `getOverdueLoans()` phát hiện có phiếu quá hạn
   - Publish `"MEMBER_LIMIT_EXCEEDED"` ngay trước khi `borrowBook` throw `MemberLimitExceededException`, để phần log/UI có thể phản ứng riêng

---

## 📌 PHASE 7 — Automated Unit Testing & Integration Flow (JUnit 5)

> **Mục tiêu**: Viết test cho đúng những nhánh nghiệp vụ dễ sai nhất **trước khi** code phần đó — làm quen với JUnit 5 (`@Test`, `assertThrows`, `assertEquals`), thứ sẽ dùng xuyên suốt khi làm Spring Boot.

### 🎯 Các bước thực hiện:

1. **Unit Test `InMemoryRepository<T>`** (dùng một class `TestEntity` giả lập để test độc lập với Book/Member/Loan):
   - `add()` rồi `findById()` phải trả đúng object vừa thêm (`assertTrue(repo.findById(id).isPresent())`)
   - `update()` chỉ thay đổi đúng field truyền vào qua `Consumer`
   - `delete()` xong thì `findById()` phải trả `Optional.empty()`

2. **Unit Test State Pattern**:
   - `AvailableState.borrow()` → `book.getStatus()` phải bằng `BookStatus.BORROWED`
   - `BorrowedState.borrow()` → `assertThrows(BookNotAvailableException.class, () -> state.borrow(book, memberId))`
   - `BorrowedState.returnBook()` → `book.getStatus()` phải chuyển về `AVAILABLE`

3. **Unit Test `LibraryService` (viết trước khi code hàm tương ứng)**:
   - Mượn sách đang `BORROWED` → `assertThrows(BookNotAvailableException.class, ...)`
   - Member đã có 5 loan active, mượn thêm → `assertThrows(MemberLimitExceededException.class, ...)`
   - Trả sách đúng hạn (`returnDate` trước hoặc bằng `dueDate`) → `assertEquals(0, fine)`
   - Trả sách trễ 3 ngày, `finePerDay = 5000` → `assertEquals(15000, fine)`
   - Trả một `loanId` đã `RETURNED` → `assertThrows(LoanAlreadyReturnedException.class, ...)`
   - `getOverdueLoans()` → chỉ chứa loan có `status == ACTIVE` và `dueDate` đã qua, không lẫn loan đã `RETURNED`

4. **Unit Test `EventPublisher`**:
   - `subscribe()` một listener rồi `publish()` → listener phải được gọi đúng 1 lần với đúng data (dùng biến đếm hoặc mock)
   - `unsubscribe()` xong rồi `publish()` lại → listener không được gọi nữa

5. **Kịch bản tích hợp hoàn chỉnh trong `Main.java`**:
   - Khởi tạo `LibraryService` ➔ thêm vài `Book` và `Member` mẫu ➔ gọi `borrowBook()` ➔ in ra Event `"BOOK_BORROWED"` nhận được ➔ giả lập ngày hiện tại vượt `dueDate` (có thể tạo `dueDate` ở quá khứ để test nhanh thay vì chờ thật) ➔ gọi `returnBook()` ➔ in ra số tiền phạt và Event `"BOOK_RETURNED"` ➔ gọi `getOverdueLoans()` để kiểm tra danh sách.

---

## 🏁 Kết quả đầu ra sau khi hoàn thành dự án này:

1. **Thành thạo Generics trong Java** (`<T extends Entity>`, bounded type, functional interface `Consumer<T>`/`Predicate<T>`): một `IRepository<T>` dùng chung được cho Book, Member, Loan.
2. **Nắm vững State Pattern kiểu Java**: dùng `switch` expression để chọn state, dùng package-private setter để khóa quyền đổi trạng thái từ bên ngoài.
3. **Biết cách viết Service điều phối nghiệp vụ phức tạp**: một hành động (mượn sách) chạm đúng vào nhiều entity liên quan, dùng `Optional` đúng chỗ thay vì kiểm tra `null` tay.
4. **Thành thạo Custom Exception (Unchecked)**: sẵn sàng viết `@RestControllerAdvice` + `@ExceptionHandler` khi chuyển sang Spring Boot — vì cơ chế bên dưới gần như giống hệt.
5. **Quen thuộc JUnit 5**: `@Test`, `assertThrows`, `assertEquals` — bộ công cụ sẽ dùng xuyên suốt trong mọi dự án Spring Boot sau này.
6. **Nắm được Observer Pattern viết tay**: hiểu rõ cơ chế bên dưới trước khi dùng `ApplicationEventPublisher`/`@EventListener` có sẵn của Spring.
