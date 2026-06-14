개발일지
=======

## 형식
    1. 프로젝트 개요
    
         본 프로젝트는 자세가 불안정할때 콘솔로 자세를 정상 주의 비정상으로 표현하기 위해 시작함
         
    2. 프로젝트 구조 & 리소스
    
         노트북이나 웹캠등 카메라로부터의 거리를 인식하기 위해 코딩을 하엿슴
        거리의 공식으로 D = {F * H}/{h} 의 공식으로 거리를 측정하엿슴
        얼굴탐지를 위해여 opencv을 사용함
    
    3. 개발 내용
    
      전 9~10 주차에서의 코드가 초록색의 물건이 있을떄에 적용이 되어 그부분을 수정함
      또한 거리간의 인식이 하나의 카메라 일때에는 제대로 측정하기 힘들어 딥러닝기반으로 짜여진 yolo을 이용하여 
      더욱더 정확한 거리측정을 하고자 연결하엿슴 또한 거리의 공식을 변화
      
    4. TMI
    
      먹은건 돈까스 정식
      
> 자료 등이 없어 첨부가 불가능한 메뉴는 기재 X

## 소스 코드
#include <opencv2/opencv.hpp>
#include <iostream>

using namespace cv;
using namespace std;

int main() {
    // 1. 카메라 열기
    VideoCapture cap(0);
    if (!cap.isOpened()) {
        cerr << "카메라를 열 수 없습니다!" << endl;
        return -1;
    }

    // 2. 사람을 인식하기 위한 HOG 디스크립터 초기화
    HOGDescriptor hog;
    // OpenCV에 기본 내장된 '보행자(사람) 인식용 SVM 모델'을 불러옵니다.
    hog.setSVMDetector(HOGDescriptor::getDefaultPeopleDetector());

    Mat frame, resized_frame;
    cout << "사람 인식 시작 (종료하려면 ESC를 누르세요)" << endl;

    while (true) {
        cap >> frame;
        if (frame.empty()) break;

        // 연산 속도를 높이기 위해 이미지 크기를 약간 줄입니다.
        resize(frame, resized_frame, Size(640, 480));

        // 사람 영역을 저장할 벡터
        vector<Rect> found_people;
        vector<double> weights;

        // 3. 사람 인식 실행 (detectMultiScale)
        // 화면 안의 사람을 찾아 found_people 벡터에 사각형 좌표를 저장합니다.
        hog.detectMultiScale(resized_frame, found_people, weights, 0, Size(8, 8), Size(32, 32), 1.05, 2);

        // 4. 인식된 사람의 위치에 빨간색 사각형 그리기
        for (size_t i = 0; i < found_people.size(); i++) {
            Rect r = found_people[i];

            // 화면에 사각형 그리기
            rectangle(resized_frame, r, Scalar(0, 0, 255), 2);

            // 텍스트 출력
            putText(resized_frame, "Person", Point(r.x, r.y - 10),
                FONT_HERSHEY_SIMPLEX, 0.6, Scalar(0, 0, 255), 2);
        }

        // 결과 화면 출력
        imshow("Human Detection (HOG)", resized_frame);

        // ESC 키를 누르면 루프 탈출
        if (waitKey(30) == 27) break;
    }

    cap.release();
    destroyAllWindows();
    return 0;
}
