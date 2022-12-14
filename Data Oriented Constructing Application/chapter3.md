## Indexing (색인)

색인은 데이터베이스의 읽기 성능을 향상시키기 위해서 등장한 개념이다.

이 작업은 데이터베이스 자체의 내용에는 관여하지는 않으나, 쓰기 과정에서 오버헤드를 발생시킨다. 이유는 아래와 같다.

- 제일 간단한 쓰기 방식은 **단순히 데이터를 append하는 것이다**.
- 그러나 인덱싱은 일반적으로 B-tree 같은 별도의 자료구조를 이용하여 쓰기 작업을 진행하기 때문에 오버헤드가 유발되는 것은 자연스러운 현상으로 여겨진다.

## 해시 색인 (Hash Indexing)

색인에 대한 대략적인 내용은 알아봤으니, 간단하게 Key-Value를 색인하는 자료구조인 해시 색인에 대해서 알아보도록 하자.

해시 색인은 HashMap을 기반으로 **Key에는 특수한 정수, Value에는 특정 데이터가 위치한 바이트 오프셋 (디스크에 위치한 데이터의 오프셋 값)**을 대응시키는 방식으로 동작한다고 가정을 해보도록하자. 그리고 이러한 해시맵은 항상 메모리에 위치한다고 가정을 하자.

이러한 경우에는 장단점이 있는데, 장점과 단점은 각각 다음과 같다.

1️⃣ 장점

- 해시맵을 모두 메모리에 적재하는 방식이기 때문에 Key를 통해서 Value 값을 찾아오는 속도가 매우 빠르다.(게다가 해시맵은 정말 최악의 경우에서도 O(logN)의 성능을 보장하기 때문에 더더욱 빠르다고 볼 수 있다.)
- 항상 해시맵에 Key-Value 쌍을 추가하는 방식으로 인덱싱이 가능하기 때문에 나름 쓰기 성능도 준수한 편에 속한다.

2️⃣ 단점

- 해시 인덱싱 정보를 항상 메모리에 적재해야하기 때문에 메모리에 해당 데이터를 유지하는데에 비용이 투입된다.
- 항상 쓰기만 일어나면 언젠가는 메모리가 가득차는 현상이 발생할 수 있다.

그렇기 때문에 Hash Indexing 방식에서는 이러한 단점을 보완하기 위해서 특정 크기의 segment로 로그를 쪼개서 보관하는 방식을 채택하는데, segment 내부의 데이터가 특정 크기에 도달하면 즉시 segment를 닫고 다른 segment를 활성화시키는 방식으로 동작시키는 것을 고려해볼 수 있다. 그리고 닫힌 segment에 대해서는 compaction algorithm을 이용해서 segment를 압축하여 저장 용량을 확보하는 정책도 생각해볼 수 있을것이다.

> 💡 **실제로 Apache Kafka가 이러한 방식을 채택하고 있는데, Topic의 데이터를 파티션 단위로 관리하여, 파티션의 데이터를 모두 segment로 관리하는데, segment 단위로 압축을 진행하거나, 혹은 일정 기간이 지난 segment에 대해서 삭제하는 정책을 선택할수도 있다.**

하지만 segment는 쓰여진 이후에 변경이 불가능하기 때문에, 압축을 진행한다면 segment 파일 자체를 조작하는 방식이 아니라, segment를 하나 새로 만들어서 해당 segment에 압축된 데이터를 넣어버리는 방식으로 진행하게 될것이다. 그리고 이러한 segment의 병합과 압축은 백그라운드 쓰레드를 이용해서 진행을 하게된다. 해당 방식에도 장단점이 있는데, 아래와 같다.

1️⃣ 장점

- 병합과 압축 과정을 통해서 인덱스 데이터를 줄이기 때문에 검색 성능이 향상되는 효과를 보인다.
- segment의 병합, 압축 과정에서 segment를 새로 쓰기 때문에 압축 과정에서 디스크의 파편화 현상을 방지할 수 있다. 즉, 데이터를 탐색하기 위해 Disk를 조회할 때 랜덤 I/O가 아닌 순차 I/O를 할 수 있기 때문에 성능이 향상되는 효과를 볼 수 있다.

2️⃣ 단점

- 병합, 압축 과정에서 쓰기 연산이 과도하게 일어나는 경우가 문제가 발생할 수 있다. 병합, 압축 과정이 백그라운드 쓰레드에서 일어난다고 할지라도 Disk의 쓰기 대역폭을 일반적인 쓰기와 공유를 해야하기 때문에 사용자가 체감하는 쓰기 성능이 떨어질 수도 있다.
- 병합, 압축만으로 메모리 상에 해시맵을 보관하기 힘들 경우, Disk에 비활성된 segment를 보관하는 방식을 채택할 수도 있는데, 이 때는 Disk 상에 존재하는 segment의 Key에 접근하는 로직이 필요하기 때문에 이 때 랜덤 I/O가 과도하게 발생할 우려가 있다. 데이터베이스에 랜덤 I/O가 과도하게 발생하는 것은 매우 안 좋은 현상이다.
- 해시테이블은 기본적으로 범위 탐색 (range query)에 취약하다. 왜냐하면 Key를 만들 때 내부적으로 데이터를 정수 타입 (보통은 int)으로 해싱을 하기 때문인데, 범위 탐색을 할 때도 탐색 대상을 해싱해서 해쉬맵을 뒤져야하기 때문에 일반적인 B-tree와는 다르게 범위 탐색에 약점을 가지고있다.

다음 주제에서는 3번째로 언급한 단점인, 범위 탐색에 대한 약점을 보완하는 인덱싱 전략에 대해서 소개해보겠다.

## SS테이블과 LSM 트리

이제 segment 파일에 한 가지 전략을 추가해보자. 변경 요구사항은 **Key-Value 쌍을 Key를 기준으로 정렬을 수행하자는 것이다**.

이처럼 키로 정렬된 Hash Index를 **SS테이블 (Sorted String Table)**이라고 부른다. 해당 SS 테이블 방식은 아래의 장점을 가진다.

- Key-Value쌍을 정렬해서 segment 파일을 보관하기 때문에, 메모리 상에 희소한 방식으로 row를 적재한다고 할지라도 희소한 데이터를 기반으로 특정 데이터의 위치를 추론하는게 가능해진다. 따라서 Disk에 접근하는 횟수를 줄이는 효과를 불러오기 때문에 쓰로풋이 향상된다.
- 백그라운드에서 압축을 진행하기 때문에 다중 세그먼트가 동일한 Key 값을 잠깐 가진다고 할지라도 압축 이후에는 오래된 segment의 Key-Value 데이터를 버리게 된다. 즉, Key에 대해서 Value를 유일하게 가지는 시점이 대부분이 된다.

그리고 이러한 SS Table을 메모리에 저장하게되면, 이러한 Hashmap을 일반적으로 **멤테이블(MemTable)**이라고 부르는데, 이러한 알고리즘을 **레벨DB, 록스DB, 카산드라, HBase**에서 사용한다고한다. 그리고 이 데이터베이스들에서 SS 테이블을 일반적으로 LSM 트리라고 부르기도한다. (물론 몇 가지의 알고리즘이 SS 테이블에 추가된 형식이다)

> 💡 **참고로 Apache Kafka가 지금까지 설명한 내용과 매우 유사한 알고리즘으로 segment를 관리하는데, 실제로도 Apache Kafka는 내부 segment를 file system에서 관리하기 위해서 록스DB를 사용하기 때문이다.**

## B-Tree

지금부터는 LSM트리 방식과 다른 데이터 색인 방식인 B-Tree 방식에 대해서 소개를 해보고자한다.

이전까지 소개한 LSM 트리 방식은 순차적으로 segment를 기록하는 방식이기 때문에 Key만 어떻게든 찾으면 그 이후로는 순차 I/O로 데이터를 찾아오는게 가능했다. 하지만 미리 스포를 해보자면, B-Tree는 그런거 없다. 디스크에 접근하면 거의 무조건 랜덤 I/O가 발생한다고 보면된다.

B-Tree는 LSM 트리 방식과는 다르게 segment 단위로 로그를 보관하는 형식이 아닌, **페이지 단위로 데이터를 보관하게 되고, 페이지의 사이즈는 전통적으로 4KB 정도로 책정이 된다**. 그리고 이러한 페이지 단위로 B-Tree Node를 보관하게 되는데, Node의 종류는 아래의 세가지가 되겠다.

- Root Node: B-Tree 자료구조의 최상단 노드. 하위 페이지의 참조를 가지고있다.
- Branch Node: B-Tree 자료구조에서 최상단 노드와 말단 노드 사이에 존재하는 페이지를 의미한다. 이 노드 또한 하위 노드의 참조를 가지고있다.
- Leaf Node: B-Tree 자료구조에서 말단 노드 역할을 수행한다. Key-Value 쌍에서 Value에 실제 데이터를 담고있거나, 혹은 실제 데이터가 위치한 디스크 페이지의 참조를 가지고있다.

참고로, B-Tree는 말단 노드 (Leaf Node)가 아니라면 하위 페이지의 참조를 가지고 있는데, 하위 페이지 참조의 최대 개수를 **분기 계수(branching factor)**라고 한다.

> 💡 **참고로, B-Tree 구조는 앵간하면 5-depth를 넘어가지 않는다고한다. 분기 계수가 일반적으로 500으로 설정이 되어있는데, 분기 계수가 500인 B-Tree에서 4-depth만 가지더라도 256TB의 데이터를 저장할 수 있다고 알려져있기 때문이다.**

그리고 B-Tree는 항상 균형을 맞추는 트리 자료구조이기 때문에, 쓰기 연산에 대해서 꽤나 높은 복잡도를 가진다는 특징을 가진다. 그리고 Leaf Node에서 페이지가 가득차게되면 Leaf Node를 쪼개게 되는데, 이 때 상위의 Branch Node에 해당 연산이 전파가 되기 때문에 이러한 특징도 B-Tree의 쓰기 복잡도를 올리는데 한 몫을 하기도한다.

이 과정에서 고아 페이지(orphan page: 부모 페이지가 존재하지 않는 페이지)가 발생할 위험이 있는데, 이 현상을 방지하기 위해서 일반적인 RDB는 재실행 로그 (Redo log)를 구현한다.

> 💡 **Redo log의 경우 고아 페이지 발생을 막기 위해 존재하는 목적도 있으나, 트랜잭션의 ACID를 보장하기 위해서라도 존재하는 목적이 존재한다.**

## B-Tree 인덱싱과 LSM 트리 인덱싱의 비교

결론부터 말하면, 경험적으로 LSM 트리는 보통 쓰기에서 더 빠르고 읽기에서 느린 특징을 보이지만, B-Tree는 쓰기 성능이 좀 느린 대신에 읽기에서 더 빠르다는 특징을 보인다.

읽기가 LSM 트리 방식에서 느린 이유는, 백그라운드에서 컴팩션 단계에 있는 segment에서 데이터를 참조해야할 경우가 자주 발생하기 때문이다.

하지만 쓰기 속도가 LSM 트리 방식에서 빠르다고 여겨지는 이유라면, B-Tree 인덱싱 방식에서는 인덱싱 데이터를 추가하기 위해서 Redo log를 만들고, 그리고 B-Tree에 데이터를 추가하는 이중 쓰기가 발생하는 반면에, LSM 트리는 일단 append하기만 하면 끝이기 때문이다. 물론 정렬은 그 이후에 Background Thread의 몫이긴하다.

그리고 LSM 트리는 일반적으로 압축률이 매우 높다. LSM 트리를 설명했을 때 말했다시피, LSM 트리는 데이터를 순차적으로 쌓는 방식이기 때문에 디스크 파편화 현상이 B-Tree에 비해서 적을 수 밖에 없다. 그리고 Segment를 병합, 압축할 때 기존의 Disk Data를 유지하면서 압축하는 방식이 아니라 segment를 새로 써버리기 때문에 디스크 파편화가 적다. 따라서 압축률이 B-Tree에 비해서 우수하다는 특징도 가진다.

하지만 LSM 트리가 B-Tree에 비해서 가지는 단점또한 존재하는데 다음과 같다.

- LSM 트리는 데이터의 유입 속도를 조절하는 알고리즘을 가지고있지 않다. 따라서 컴팩션 과정에서 압축되는 속도를 유입 데이터가 압도하는 순간이 오래 지속되면 메모리가 가득찰 때 까지 계속 데이터를 쌓게된다.
- 백그라운드의 컴팩션 과정이 때때로 일반적인 읽기/쓰기 성능에 영향을 미칠 때가 있다. 압축중인 segment를 대상으로 IO가 일어날 때 문제가 생긴다.
- Key의 유일성이 B-Tree에 비해서 보장이 잘 되지않는다. compaction이 일어나지 않은 순간에는 다중 key의 복사본이 존재할 수 있기 때문이다.

그리고 B-Tree가 LSM 트리에 비해서 가지는 장점이 있는데, 다음과 같다.

- Key의 유일성이 보장이 무조건적으로 되는건 아닌데, B-Tree에서 각 키가 색인 한 곳에서만 존재한다는게 보장이 된다. 즉, Key의 값이 동일하더라도 색인의 한 곳에만 독립적으로 존재할 수 있을 뿐더러, 복사본의 존재성이 차단된다.
- 많은 관계형 데이터베이스에서는 B-Tree에서 Key-Value 단위의 레코드 잠금 기능을 지원한다. 이를 이용해서 트랜잭션의 ACID를 보장하거나, 혹은 트랜잭션의 격리수준 이라는 기능을 제공하기도한다. 하지만 LSM 트리를 구현하는 데이터베이스는 그런거 없다.

## Clustered Index, Non-Clustered Index(Secondary Index)

다음으로 소개할 내용은, 군집형 인덱스와 비군집형 인덱스이다.

이 둘의 내용은 B-Tree, LSM 트리 방식과는 무관한 방식임을 미리 알아두자.

색인은 지금까지 설명했다시피 Key에 대응하는 Value의 타입은 row에 대한 참조이거나, 혹은 row 자체를 가지고있는 형태일 수도 있다고 설명한 바 있다.

여기서 Value가 직접 row의 내용을 가진 형태를 Clustered Index라고 부르는데, 이러한 경우에는 Index를 탐색하고 Disk에 Random IO를 일으키는 양을 줄여줄 수 있기 때문에 읽기 속도가 매우 향상되는 효과를 불러온다. 반대로 Index 자체가 큰 용량을 요구하는 상황이 일어날 수 있기 때문에 인덱스가 차지하는 용량이 커지는 효과를 불러오기도 한다.

Clustered Index는 일반적으로 RDB에서 Primary Key를 할당하면 PK를 보관하는 인덱스가 Clustered Index로 구현이 된다.

Non-Clustered Index는 이와 반대인데, Key에 대해서 데이터의 참조를 가진다. 이는 일반적으로 FK의 구현에 이용이 된다.

> 💡 **RDB에서는 외래키 참조 무결성 제약이란게 존재하는데, 부모의 데이터가 변경될 시 해당 참조를 가지는 Secondary Index의 Key-Value 쌍들에 잠금이 전파되는 현상이 관측될 수 있다. 이로 인해서 데드락이 벌어질 수도 있기 때문에, 실무에서는 이러한 잠금 전파 현상을 막기 위해서 FK를 맺지 않는 것을 관측할 수 있다.**

그리고 이 둘 간의 절충안도 존재하는데, Key에 테이블의 일부 컬럼만 저장하는 방식으로 인덱싱을 할 수도 있는데, 이러한 인덱싱 기법을 **커버링 인덱스(Covering Index)**라고 부른다.