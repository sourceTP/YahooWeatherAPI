
시간의 개발팀에서 다음의 문제를 함께 해결하기 위해 요청하고 해결된 문제에 대해 공유 합니다.

####1. 배경
- 시간의 날씨정보 API를 기존에 SK Planet에서 사용하던 중 서비스가 중지되어 다른 대안을 찾던 중
  Yahoo에서 제공하는 현재 날씨 API로 변경
- 이때 모바일 디바이스의 GPS 정보만으로 주소를 확인 해당 주소지의 날씨를 Yahoo에 의뢰 그 결과를 확인
- GSP정보만으로 주소를 확인해주는 서비스가 느리거나 자주 요청될 경우 요청이 응답을 하지 않음
- 때문에 행정안전부에서 제공하는 도로명 주소기준의 데이터 가져와 우리 DB에서 직접 조회하는
  방법을 사용하고자 6백 30만건 정도의 데이터를 우리 DB에 추가 하였음

####2. 문제
- 준비된 데이터 셋 : 도로명 주소 위치(X,Y)
    . 주소 Record + X 좌표 + Y 좌표 > 여기서 좌표 값은 EPSG 5178 포맷
    . 국내 주소지기준으로 좌표를 표시할 수 있는 6백 30만건의 데이터
- 요청된 데이터 셋 : 디바이스의 위치(위도,경도)

요청된 디바이스의 위치를 데이터베이스 주소 정보의 위치에서 가장 근접한 위치의 주소지를 찾는 것
* EPSG 5178 > 4326 > 5178 로의 변환은 문제 없음.

####3. 해결방안-현재
- 절차
   1) 디바이스에서 요청된 위경도 기반의 데이터(4326)를 도로명 주소지 위치(5178)로 변경
   2) 변경된 위치를 도로명 주소지 위치에서 X 축과 가장 근접하고 Y 축과 가장 근접한 주소를 찾음
   3) 해당 위치의 주소지 정보를 Yahoo 날씨 정보 확인

- 데이터
<code>
   AddressCoordinate = [{‘x’: 39.7612992, ‘y’: -86.1519681},
                        {‘x’: 39.762241,  ‘y’: -86.158436 },
                        {‘x’: 39.7622292, ‘y’: -86.1578917}]
  DeviceCoordinate = {'lat': 39.7622290, 'lon': -86.1519750}

  * AddressCoordinate 목록에서 DeviceCoordinate 가 가장 근접한 목록을 찾는 방법
</code>

- 코드
<code>
      from math import cos, asin, sqrt

      def distance(lat1, lon1, lat2, lon2):
            p = 0.017453292519943295
            a = 0.5 - cos((lat2-lat1)*p)/2 + cos(lat1*p)*cos(lat2*p) * (1-cos((lon2-lon1)*p)) / 2
            return 12742 * asin(sqrt(a))

      def closest(data, v):
            return min(data, key=lambda p: distance(v['lat'],v['lon'],p['lat'],p['lon']))

      AddressCoordinates = [{‘x’: 39.7612992, ‘y’: -86.1519681},
                                 {‘x’: 39.762241,  ‘y’: -86.158436 },
                                 {‘x’: 39.7622292, ‘y’: -86.1578917}]
      DeviceCoordinate = {'lat': 39.7622290, 'lon': -86.1519750}

      print(closest(AddressCoordinates, DeviceCoordinate))
      # AddressCoordinates 는 요청된 디바이스 위치 정보에서 후보 ZONE을 구성해
      # 100 이내로 가져와 구성 (위도,경도,위도+3,경도+3)
</code>

