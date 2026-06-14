개발일지
=======

## 형식
    1. 프로젝트 개요

        본 프로젝트는 자세가 불안정할때 콘솔로 정상 주의 비정상으로 만들기 위해 시작하엿슴

    2. 프로젝트 구조 & 리소스

        노트북이나 웹캠등 카메라로부터의 거리를 인식하기 위해 코딩을 하엿슴
        거리의 공식으로 Distance = {W * f}/{w} 의 공식으로 거리를 측정하엿슴
        얼굴탐지를 위해여 opencv을 사용함

    3. 개발 내용

        본아래 코딩을 참조함

    4. TMI
        먹은것은 콘푸라이트
> 자료 등이 없어 첨부가 불가능한 메뉴는 기재 X

## 소스 코드
#include <opencv2/opencv.hpp>
#include <iostream>

using namespace cv;
using namespace std;

int main() {
    // 1. 상수 설정 (실제 환경에 맞게 수정 필요)
    const double REAL_WIDTH = 14.0;    // 물체의 실제 가로 길이 (예: 14cm)
    const double FOCAL_LENGTH = 700.0; // 사전 테스트로 구한 카메라의 초점 거리 (픽셀 단위)

    // 웹캠 열기 (기본 카메라: 0)
    VideoCapture cap(0);
    if (!cap.isOpened()) {
        cerr << "카메라를 열 수 없습니다!" << endl;
        return -1;
    }

    Mat frame, hsv, mask;

    cout << "거리 측정 시작 (종료하려면 ESC를 누르세요)" << endl;

    while (true) {
        cap >> frame;
        if (frame.empty()) break;

        // 2. 이미지 처리: 특정 색상(초록색) 영역 추출
        cvtColor(frame, hsv, COLOR_BGR2HSV);
        // HSV 색상 공간에서 초록색 범위 지정
        inRange(hsv, Scalar(35, 100, 100), Scalar(85, 255, 255), mask);

        // 노이즈 제거
        erode(mask, mask, Mat());
        dilate(mask, mask, Mat());

        // 3. 윤곽선(Contour) 검출
        vector<vector<Point>> contours;
        findContours(mask, contours, RETR_EXTERNAL, CHAIN_APPROX_SIMPLE);

        for (const auto& contour : contours) {
            // 노이즈 방지를 위해 일정 크기 이상의 윤곽선만 처리
            if (contourArea(contour) > 1000) {
                // 물체를 감싸는 사각형(Bounding Box) 구하기
                Rect bbox = boundingRect(contour);
                int pixel_width = bbox.width; // 화면 속 물체의 픽셀 가로 크기

                // 4. 거리 계산 공식 적용
                double distance = (REAL_WIDTH * FOCAL_LENGTH) / pixel_width;

                // 5. 화면에 시각화
                rectangle(frame, bbox, Scalar(0, 255, 0), 2); // 물체에 초록색 사각형 그리기

                string text = "Distance: " + to_string(distance) + " cm";
                putText(frame, text, Point(bbox.x, bbox.y - 10),
                    FONT_HERSHEY_SIMPLEX, 0.6, Scalar(0, 255, 0), 2);
            }
        }

        // 결과 화면 출력
        imshow("Distance Estimation", frame);

        // ESC 키를 누르면 루프 탈출
        if (waitKey(30) == 27) break;
    }

    cap.release();
    destroyAllWindows();
    return 0;
}


- - -
