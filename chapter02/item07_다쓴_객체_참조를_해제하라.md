# ITEM07.다 쓴 객체참조를 해제하라

---
### 다 쓴 객체 참조를 해제해야 하는 이유
- 객체 참조 하나를 살려두면 GC는 그 객체뿐만 아니라 그 객체가 참조하는 모든 객체를 회수하지 못할 수 있다.
- 이로 인해 메모리 누수가 발생하며 성능 저하 발생으로 이어진다.

---
### 메모리 누수의 대표 원인과 대응

#### 1. 자기 메모리를 직접 관리하는 클래스
- 문제 : 배열 기반 컨테이너에서 비활성 영역이 여전히 객체를 참조한다.
    ```
    //스택에서 꺼낸 원소가 배열에 남아있는 경우
    public Object pop() {
        ...
        return elements[--size];
        ...
    }
    ```
  - 스택에서 꺼낸 객체는 사실상 **다 쓴 참조(obsolete reference)**이다.
  - 하지만 배열은 여전히 그 객체를 가리키므로, GC 입장에서는 size 이상 인덱스도 여전히 유효한 참조로 보인다.
  - 따라서 GC는 이를 회수하지 않는다.
- 해결 방안 : 원소 제거 시 `null` 처리를 해서 해당 객체를 더는 쓰지 않을 것임을 GC 에게 알려야 한다.
    ```
    public Object pop() {
        ...
        Object result = elements[--size];
        elements[size] = null; //비활성 영역 참조 해제
        return result;
        ...
    }
    ```
  - 이점: 잘못된 접근 시 `NullPointerException`을 던져 조기 실패를 파악할 수 있다.
  - 단, 무분별한 null 남발은 금물이며, 변수 스코프를 최소화하면 참조는 자연스럽게 소멸된다. (아이템 57)

#### 2. 캐시
- 문제 : 객체 참조를 캐시에 넣고 다 쓴 뒤에도 놔두면 메모리 누수가 발생한다.
- 해결 방안
1) `WeakHashMap` 사용
    </br> 캐시 외부에서 key 를 참조하는 동안만 엔트리가 살아있게 관리하여 다쓴 엔트리는 즉시 자동 제거된다.
    ```
    public static void main(String[] args) {
        final WeakHashMap<String, Inteager> cacheMap = new WeakHashMap<>();
        String name1 = "Kang";
        String name2 = "Lee";
        map.put(name1, 27);
        map.put(name2, 25);
            
        name1 = null;
        System.gc();
            
        map.entrySet().stream().forEach(entry -> System.out.println(entry)); // Lee = 25
    }

2) 만료 정책 추가</br>
캐시 엔트리의 유효기간을 정의하기 어렵다면, 시간이 지남에 따라 엔트리의 가치를 떨어뜨리는 방식을 사용한다
   - 스케줄러 활용 (예 : `ScheduledThreadPoolExecutor`)
   - 부수 작업 활용 (예 : `LinkedHashMap.removeEldestEntry` -> LRU 캐시)
    ```
    Map<String, byte[]> lru = new LinkedHashMap<>(16, 0.75f, true) {
        private static final int MAX_ENTRIES = 1000;
        @Override
        protected boolean removeEldestEntry(Map.Entry<String, byte[]> eldest) {
            return size() > MAX_ENTRIES;
        }
    };
    ```
3) 더 복잡한 캐시를 원한다면 java.lang.ref 패키지(SoftReference, WeakReference 등)를 직접 활용한다.

#### 3. 리스너 혹은 콜백
- 문제 : 클라이언트가 콜백을 등록만 하고 명확히 해지하지 않는다면, 콜백은 계속 쌓여가서 메모리 누수가 발생할 수 있다.
- 해결 방안
  - 해지 API 를 제공하고 사용 시점에 반드시 호출
  - 또는 약한 참조(WeakReference) 로 보관해 누수 방지
  ```
    public class SomeHandler extends Handler {
    private final WeakReference<SomeManager> mSomeManager;
    
            public SomeHandler(SomeManager someManager, Looper looper) {
                super(looper);
                someManager = new WeakReference<>(someManager);
            }
    
            @Override
            public void handleMessage(@NonNull Message msg) {
                final SomeManager someManager = someManager.get();
                if (someManager == null) {
                    Log.w(TAG, "handleMessage :: SomeManager garbage collected, return.");
                    return;
                }
                someManager.processForSomething((byte[]) msg.obj, msg.arg1, msg.arg2);
            }
        }

### Weak Reference
- Java 1.2부터 java.lang.ref 패키지를 통해 객체 참조를 strong / soft / weak / phantom으로 구분할 수 있다.- 이 중 weak reachable 객체인 Weak Reference 는 GC가 동작할 떄 무조건 수거되는 성격을 갖고 있다. 
- 그 중 Weak Reference 는 GC 시 무조건 수거되는 성격을 갖는다.

#### 생성 및 사용
```
// 생성
WeakReference<SomeObject> someObj = new WeakReference<SomeObject>(new SomeObject());
// 사용
someObj.get();
```
### 예시
#### Weak Reference 를 사용하기 전
```
public class Image {
        public final String name;

        public Image(String name) {
            this.name = name;
        }

        @Override
        protected void finalize() throws Throwable {
            super.finalize();
            System.out.println("finalize " + name);
        }
    }

    public class ImageViewer {
        public final Image image;

        public ImageViewer(Image image) {
            this.image = image;
        }
    }
```
- Image 를 null 로 만들어 참조를 해제했다고 할 때 System.gc() 를 호출했음에도 불구하고 Image 객체의 fianlize는 호출되지 않는다.
- ImageViewer 가 Image 객체를 들고 있기 때문에 Image 가 아직 reachable 상태이기 때문이다.
#### WeakReferece 를 사용 후
```
    public class ImageViewer {
        public final WeakReference<Image> image;

        public ImageViewer(Image image) {
            this.image = new WeakReference<>(image);
        }
    }

```
- Image 객체의 참조를 해제하여 현재 Image 객체가 참조되고 있는 곳이 ImageViewer 의 WeakReference 뿐이라면 GC 수행시에 Image 객체의 메모리는 해제되게 된다.
- GC는 unreachable 객체뿐만 아니라 weakly reachable 객체도 가비지 객체로 간주되어 메모리에서 회수되기 떄문이다.
- 즉, Root set 으로부터 참조 사슬에 속해 있더라도 개발자 요구에따라 객체가 GC 의 대상이 될 수 있다.