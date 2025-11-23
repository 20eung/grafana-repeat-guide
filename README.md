# Grafana 패널 자동화 가이드: Variables와 Repeat Options 활용

이 문서는 Grafana에서 장비나 서버가 늘어날 때마다 패널을 일일이 만들지 않고, **변수(Variable)**를 활용하여 선택한 장비 수만큼 패널을 **자동으로 생성(Repeat)**하는 방법을 설명합니다.

## 1. 변수(Variable) 생성하기

가장 먼저 하드코딩된 장비 이름 대신 사용할 변수를 생성해야 합니다.

1.  대시보드 상단 **Settings (톱니바퀴 아이콘)** > **Variables** 메뉴로 이동합니다.
2.  **New variable**을 클릭하고 아래와 같이 설정합니다.

| 설정 항목 | 입력값 | 설명 |
| :--- | :--- | :--- |
| **Variable type** | `Query` | DB에서 값을 가져옴 |
| **Name** | `host` | 쿼리에서 사용할 변수명 (예: `$host`) |
| **Label** | `장비명` | 표시되는 이름 (선택사항) |
| **Show on dashboard** | `Label and value` | 기본값 |
| **Data source** | `InfluxDB` | 사용 중인 데이터 소스 선택 |
| **Query** | `SHOW TAG VALUES WITH KEY = "hostname"` | 장비 이름(Tag) 목록을 가져오는 쿼리 |
| **Sort** | `Alphabetical (asc)` | 가나다순 정렬 (선택사항) |
| **Refresh** | `On dashboard load` | 대시보드 로딩 시 목록 갱신 |

### ✅ 중요 체크 옵션 (Selection Options)
아래 옵션을 반드시 체크해야 여러 장비를 동시에 선택하거나 전체를 볼 수 있습니다.
* [x] **Multi-value**: 여러 장비 동시 선택 가능
* [x] **Include All option**: 'All' 선택 시 모든 장비 표시

---

## 2. 패널 쿼리(Query) 수정하기

생성한 변수(`$host`)를 패널 쿼리에 적용합니다.

1.  패널의 **Edit** 모드로 진입합니다.
2.  `WHERE` 절의 고정된 장비 이름을 변수로 변경합니다.

**변경 전 (하드코딩):**
```sql
SELECT non_negative_derivative("ifHCOutOctets", 1s) *8 AS "bandwidth"
FROM "snmp" 
WHERE  "hostname" = "Router_01"
AND "ifDescr" !~ /BB|TMS/ AND "ifDescr" !~ /^p/
AND $timeFilter 
GROUP BY "hostname", "ifDescr", "ifAlias"
```

**변경 후 (변수 적용):**
```sql
SELECT non_negative_derivative("ifHCOutOctets", 1s) *8 AS "bandwidth"
FROM "snmp" 
WHERE  "hostname" =~ /^$host$/
AND "ifDescr" !~ /BB|TMS/ AND "ifDescr" !~ /^p/
AND $timeFilter 
GROUP BY "hostname", "ifDescr", "ifAlias"
```

> **💡 중요 팁:**
> * `=` 대신 `=~` 연산자를 사용해야 정규표현식 매칭이 되어 **Multi-value(다중 선택)**가 작동합니다.
> * `/^$host$/`로 감싸주는 것이 안정적입니다.

---

## 3. 패널 제목 동적화 (선택사항)

패널이 여러 개로 복사될 때, 각 패널이 어떤 장비인지 알 수 있도록 제목도 변수로 설정합니다.

* **Panel options** > **Title** 입력란에 아래와 같이 입력합니다.
    > `$host 트래픽 현황`
* **결과:** 드롭다운에서 선택한 장비 이름에 따라 "Router_01 트래픽 현황", "Router_02 트래픽 현황"으로 자동 변경됩니다.

---

## 4. Repeat Options (자동 반복) 설정

이 기능이 핵심입니다. 변수에서 선택된 값의 개수만큼 패널을 복제합니다.

1. 패널 편집 화면 우측 사이드바에서 **Panel options** 섹션을 펼칩니다.
2. **Repeat options**를 찾아 아래와 같이 설정합니다.

| 옵션 | 설정값 | 설명 |
| :--- | :--- | :--- |
| **Repeat by variable** | `host` | 위에서 만든 변수 선택 |
| **Repeat direction** | `Horizontal` | 패널을 가로로 나열 (Vertical은 세로) |
| **Max per row** | `2` | (Horizontal 선택 시) 한 줄에 보여줄 패널 개수 |

---

## 5. 적용 결과 확인

1.  대시보드를 **Save** 하고 나옵니다.
2.  상단의 **`host`** 드롭다운 메뉴에서:
    * 장비를 2개 이상 체크해 봅니다.
    * 또는 **`All`**을 선택해 봅니다.
3.  선택한 장비의 개수만큼 패널이 자동으로 생성되어 배치되는지 확인합니다.
