# 🌧️ KMA ASOS 강수량 템플릿 (Flutter Web/App 안전 버전)

광주·전남 지역 누적 강수량을 기상청 ASOS 시간자료(HR) API로 조회하는 예제 템플릿입니다. 웹과 모바일 앱(Android/iOS)을 동시에 지원하며, HTTPS·프록시 분기, API 키 저장, 강수량 누적 계산 로직을 한눈에 확인할 수 있습니다.

## 🙋 처음 쓰는 분들을 위한 쉬운 설명
1. **파일 받기**: 이 페이지 오른쪽 위의 초록색 `Code` 버튼 → `Download ZIP`을 눌러 압축파일을 받습니다. (GitHub를 몰라도 됩니다.)
2. **압축 풀기**: 받은 ZIP을 원하는 폴더에 압축 해제합니다. 폴더 이름은 `kma_rainfall`이어도 괜찮습니다.
3. **편집기 열기**: VS Code, Android Studio, IntelliJ 등 익숙한 에디터에서 해당 폴더를 엽니다.
4. **API 키 저장**: Flutter 앱 안에서 `SharedPreferences`에 `kma_api_key`라는 이름으로 기상청 서비스키를 한 번 저장해 둡니다. (별도 UI가 없다면 `SharedPreferences.setString('kma_api_key', '서비스키')` 코드를 임시로 실행하세요.)
5. **프록시 주소 지정**: 웹에서 쓸 리버스 프록시 도메인을 가지고 있다면 `lib/services/kma_api_service.dart` 상단의 `proxyHost` 값을 `api.your-domain.kr`처럼 실제 주소로 바꿉니다. 없다면 기본값을 두고 모바일 앱에서만 먼저 테스트해도 됩니다.
6. **실행하기**: 터미널을 열어 `flutter pub get` → `flutter run -d chrome`(웹) 또는 `flutter run`(앱)을 실행하면 예제 코드에서 누적 강수량을 불러올 수 있습니다.
7. **URL이 왜 없나요?**: 이 템플릿은 소스코드만 제공하므로, 직접 Flutter 웹을 빌드(`flutter build web`)해서 원하는 서버나 호스팅 서비스에 올려야만 접속 가능한 URL이 생깁니다. README의 Nginx 예시는 그런 배포 환경을 만드는 방법을 보여주는 예시입니다.

> 한 줄 요약: ZIP으로 받아 Flutter 프로젝트로 열고, API 키/프록시만 채워서 직접 실행·배포하면 됩니다. 따로 제공되는 “확인용 URL”은 없습니다.

## 📦 폴더 구조
```
kma_rainfall_template/
├─ README.md
└─ lib/
   └─ services/
      ├─ kma_api_service.dart       # HTTPS + 자동 인코딩 + 프록시 분기
      └─ rainfall_service.dart      # 누적 강수량 계산 (SharedPreferences 저장)
```

## 🚀 핵심 기능
- `Uri.https()`를 사용해 파라미터를 안전하게 자동 인코딩합니다.
- 웹은 `kIsWeb` 감지 후 리버스 프록시(예: `/kma/`)를 경유하고, 앱은 `apis.data.go.kr`에 직접 연결합니다.
- 시간자료 `rn` 값을 합산하여 누적 강수량(mm)을 계산합니다.
- `SharedPreferences('kma_api_key')`에 API 키를 저장한 뒤 재사용합니다.

## 🧭 요청 파라미터 (ASOS HR)
| 파라미터 | 설명 |
| --- | --- |
| `dataType` | `JSON` 고정 |
| `dataCd` | `ASOS` |
| `dateCd` | `HR` |
| `startDt` | 조회 시작 날짜(YYYYMMDD) |
| `startHh` | 조회 시작 시각(HH) |
| `endDt` | 조회 종료 날짜(YYYYMMDD, “오늘”이면 전일 D-1 권장) |
| `endHh` | 조회 종료 시각(HH) |
| `stnIds` | 관측소 ID (광주 156 등) |

> **Tip**: 실시간 “오늘” 조회 시 `endDt`를 전일(D-1)로 보정해야 데이터가 누락되지 않습니다.

## 🧰 서비스 코드
### `lib/services/kma_api_service.dart`
```dart
import 'dart:convert';
import 'package:flutter/foundation.dart' show kIsWeb;
import 'package:http/http.dart' as http;

class KmaApiService {
  static const String proxyHost = 'REPLACE_WITH_PROXY_HOST'; // 웹 프록시 도메인
  static const String _originHost = 'apis.data.go.kr';
  static const String _asosPath = '/1360000/AsosHourlyInfoService/getWthrDataList';

  Future<Map<String, dynamic>> getAsosHourlyJson({
    required String serviceKey,
    required String stnIds,
    required String startDt,
    required String startHh,
    required String endDt,
    required String endHh,
    int pageNo = 1,
    int numOfRows = 999,
  }) async {
    final query = {
      'serviceKey': serviceKey,
      'pageNo': '$pageNo',
      'numOfRows': '$numOfRows',
      'dataType': 'JSON',
      'dataCd': 'ASOS',
      'dateCd': 'HR',
      'startDt': startDt,
      'startHh': startHh,
      'endDt': endDt,
      'endHh': endHh,
      'stnIds': stnIds,
    };

    final host = kIsWeb && proxyHost != 'REPLACE_WITH_PROXY_HOST' ? proxyHost : _originHost;
    final path = kIsWeb && proxyHost != 'REPLACE_WITH_PROXY_HOST' ? '/kma$_asosPath' : _asosPath;

    final uri = Uri.https(host, path, query);
    final resp = await http.get(uri);
    if (resp.statusCode != 200) {
      throw Exception('ASOS request failed: ${resp.statusCode} ${resp.reasonPhrase}');
    }
    return json.decode(resp.body) as Map<String, dynamic>;
  }

  Future<double> getAccumulatedRainfallMm({
    required String serviceKey,
    required String stnIds,
    required String startDt,
    required String startHh,
    required String endDt,
    required String endHh,
  }) async {
    final data = await getAsosHourlyJson(
      serviceKey: serviceKey,
      stnIds: stnIds,
      startDt: startDt,
      startHh: startHh,
      endDt: endDt,
      endHh: endHh,
    );

    final response = data['response'] as Map<String, dynamic>?;
    final body = response?['body'] as Map<String, dynamic>?;
    final items = body?['items'] as Map<String, dynamic>?;
    final list = items?['item'] as List?;
    if (list == null || list.isEmpty) return 0.0;

    double total = 0.0;
    for (final e in list) {
      final m = (e as Map).cast<String, dynamic>();
      final rnRaw = m['rn'];
      total += _toDoubleSafe(rnRaw);
    }
    return total;
  }

  double _toDoubleSafe(dynamic v) {
    if (v == null) return 0.0;
    if (v is num) return v.toDouble();
    final s = v.toString().trim();
    if (s.isEmpty || s == '-' || s.toLowerCase() == 'null') return 0.0;
    return double.tryParse(s) ?? 0.0;
  }
}
```

### `lib/services/rainfall_service.dart`
```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'kma_api_service.dart';

class RainfallService {
  final KmaApiService _api;
  RainfallService(this._api);

  Future<bool> hasApiKey() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final key = prefs.getString('kma_api_key');
      return key != null && key.trim().isNotEmpty;
    } catch (_) {
      return false;
    }
  }

  Future<double> getAccumulatedRainfallMm({
    required String stnIds,
    required String startDt,
    required String startHh,
    required String endDt,
    required String endHh,
  }) async {
    final prefs = await SharedPreferences.getInstance();
    final apiKey = prefs.getString('kma_api_key')?.trim();
    if (apiKey == null || apiKey.isEmpty) {
      throw Exception('KMA API key is not set in SharedPreferences (kma_api_key).');
    }

    return _api.getAccumulatedRainfallMm(
      serviceKey: apiKey,
      stnIds: stnIds,
      startDt: startDt,
      startHh: startHh,
      endDt: endDt,
      endHh: endHh,
    );
  }
}
```

## 🌍 웹 프록시 설정 (Nginx)
```nginx
server {
  listen 443 ssl;
  server_name api.your-domain.kr;

  location /kma/ {
    proxy_pass https://apis.data.go.kr/;
    proxy_set_header Host apis.data.go.kr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    add_header Access-Control-Allow-Origin "*" always;
    add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
    add_header Access-Control-Allow-Headers "*" always;

    if ($request_method = OPTIONS) {
      return 204;
    }
  }
}
```
> 코드의 `proxyHost`를 `api.your-domain.kr`로, 경로는 `/kma/`로 일치시켜야 합니다.

## 🧪 빠른 테스트 (`main.dart`)
```dart
final api = KmaApiService();
final rainfallService = RainfallService(api);

final total = await rainfallService.getAccumulatedRainfallMm(
  stnIds: '156',      // 광주
  startDt: '20251001',
  startHh: '00',
  endDt: '20251010',  // D-1 권장
  endHh: '23',
);
print('누적 강수량: $total mm');
```

## ✅ 체크리스트
- 웹: **HTTPS** + **프록시** 설정 여부
- `serviceKey`는 설정 화면에서 저장·검증
- `endDt`는 D-1 보정
- 빈값/결측(`-`)은 0mm 처리

## 🛠️ 트러블슈팅 가이드
| 증상 | 원인 | 해결 방법 |
| --- | --- | --- |
| `ASOS request failed: 401` | 잘못된 또는 미입력 API 키 | `SharedPreferences`에 저장된 `kma_api_key`를 확인하고, UTF-8 인코딩된 서비스 키를 재발급 후 적용합니다. |
| 브라우저 콘솔 `CORS` 에러 | 프록시 미구현 또는 HTTP 호출 | 위 Nginx 예시처럼 `/kma/` 리버스 프록시를 구성하고 Flutter 웹 빌드에 `proxyHost`를 설정합니다. |
| 데이터가 0mm로 나옴 | `endDt`가 오늘 날짜로 지정 | 실시간 조회 시 종료 날짜를 전일(D-1)로 설정하거나, 기상청 데이터 업데이트 시간을 확인합니다. |
| `FormatException` 또는 `double.parse` 실패 | 결측값(`-`, `null`) 존재 | `_toDoubleSafe`처럼 문자열을 필터링한 뒤 `double.tryParse`를 사용해 변환합니다. |

## 📎 라이선스
필요 시 MIT 라이선스 문구를 추가해 사용할 수 있습니다.
