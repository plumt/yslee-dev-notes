# image 그룹 (image-picker, image-saver, share)

비트맵 입출력 세 모듈. 서로 참조하지 않으며 조합은 앱이 한다(qrscanner 가 셋을 모두 채택).

## image-picker

- **목적**: 시스템 갤러리에서 이미지를 골라 긴 변 약 1024px 이하 Bitmap 으로 돌려주는 컴포저블 하나.
- **기술**: Photo Picker(`PickVisualMedia`)이므로 저장소 권한이 필요 없다. 매니페스트에 갤러리 읽기 권한을 넣지 않는다.
- **구현**: `Dispatchers.IO` 에서 `inJustDecodeBounds` 로 크기만 읽고, 2의 거듭제곱 `inSampleSize` 로 표본화해 디코딩한 뒤 메인에서 콜백을 한 번 부른다. 취소와 실패는 모두 null 로 알린다. 콜백 본문의 예외는 호출부가 잡아야 한다.

## image-saver

- **목적**: Bitmap 을 MediaStore 로 갤러리에 PNG 로 저장하고 성공 여부(Boolean)를 돌려준다.
- **구현**: minSdk 29 라 MediaStore 경로 하나만 둔다. `IS_PENDING=1` 로 삽입, PNG 압축, `IS_PENDING=0` 으로 공개하며, 실패 갈래에서는 대기 항목을 지우고 false 를 돌려준다. 저장 위치는 `Pictures/{subDirectory}`, 파일명은 `{prefix}_yyyyMMdd_HHmmss.png`. 무상태 `object` 이며 워커 스레드에서 부르고, 코루틴에서는 `SaveImageToGalleryUseCase`(IO 전환)를 쓴다.
- **의존**: `core:fundamental:common`(api, `TimeProvider`), `kotlinx-coroutines-android`.

## share

- **목적**: Bitmap 을 캐시에 쓰고 FileProvider content URI 로 바꿔 공유 인텐트를 만든다.
- **구현**: 캐시 아래 `shared` 디렉터리에 `shared_image_<밀리초>.png` 로 쓰고 읽기 권한 플래그를 붙인 전송 인텐트를 만든다. 제목이 있으면 chooser 로 감싼다. 입출력 실패는 모두 null 로 접힌다. authority 는 `${applicationId}.share.fileprovider` 이며 코드와 매니페스트를 함께 바꿔야 한다.
- **의존**: 없음(AndroidX 만 사용).
