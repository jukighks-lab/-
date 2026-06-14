개발일지
=======

## 형식
    1. 프로젝트 개요

       본프로젝트 개요는 동일함

    2. 프로젝트 구조 & 리소스

      이하동일

    3. 개발 내용

    염동호 팀원이 mediapipe 프로그램을 통해 딥러닝 프로그램으로 구현하고자 함

    4. TMI
        이하생략
> 자료 등이 없어 첨부가 불가능한 메뉴는 기재 X

## 소스 코드
#include <opencv2/opencv.hpp>
#include <opencv2/core/utils/logger.hpp>
#include <windows.h>
#include <chrono>
#include <deque>

using namespace cv;
using namespace std;
using namespace chrono;

// ===== 상수 =====
const double FOCAL  = 700.0, FACE_W = 15.0;
const double SAFE   = 50.0,  WARN   = 40.0;
const int    SMOOTH = 5;

// ===== 거리 평활화 =====
double smoothDist(deque<double>& hist, double raw)
{
    hist.push_back(raw);
    if ((int)hist.size() > SMOOTH) hist.pop_front();
    double sum = 0;
    for (double d : hist) sum += d;
    return sum / hist.size();
}

// ===== UI 텍스트 출력 =====
void putMsg(Mat& frame, const string& msg, Point pos, Scalar color, double scale = 1.0)
{
    putText(frame, msg, pos, FONT_HERSHEY_SIMPLEX, scale, color, scale > 0.8 ? 3 : 2);
}

// ===== 자세 판정 + 경보 =====
void judgePosture(Mat& frame, double dist, bool& tooClose,
                  steady_clock::time_point& t0,
                  steady_clock::time_point& tBeep)
{
    auto now = steady_clock::now();
    double el     = duration_cast<seconds>(now - t0).count();
    double beepEl = duration_cast<seconds>(now - tBeep).count();

    if (dist >= SAFE)
    {
        putMsg(frame, "GOOD POSTURE", {30, 70}, {0, 255, 0});
        tooClose = false;
    }
    else if (dist >= WARN)
    {
        putMsg(frame, "TOO CLOSE",           {30,  70}, {0, 255, 255});
        putMsg(frame, "Please Keep Distance", {30, 110}, {0, 255, 255}, 0.8);
        tooClose = false;
    }
    else
    {
        if (!tooClose) { tooClose = true; t0 = tBeep = now; el = 0; }

        if (el < 3)
            putMsg(frame, "Warning in " + to_string(3 - (int)el) + "s",
                   {30, 70}, {0, 255, 255});
        else
        {
            putMsg(frame, "FORWARD HEAD POSTURE!", {30,  70}, {0, 0, 255});
            putMsg(frame, "Please Sit Further Back", {30, 110}, {0, 0, 255}, 0.8);
            if (beepEl >= 5) { Beep(1000, 500); tBeep = now; }  // 5초마다 반복 경보
        }
    }
}

// ===== 초점거리 캘리브레이션 =====
double calibrate(double knownDist, double pixelWidth)
{
    return (pixelWidth * knownDist) / FACE_W;
}

// ===== main =====
int main()
{
    utils::logging::setLogLevel(utils::logging::LOG_LEVEL_ERROR);

    CascadeClassifier cascade;
    VideoCapture cap(0, CAP_DSHOW);

    if (!cascade.load("haarcascade_frontalface_default.xml") || !cap.isOpened())
        return -1;

    cap.set(CAP_PROP_FRAME_WIDTH,  640);
    cap.set(CAP_PROP_FRAME_HEIGHT, 480);

    // 캘리브레이션: 50cm 거리에서 얼굴 폭(픽셀) 측정 후 자동 계산
    // double focalLen = calibrate(50.0, 측정된_픽셀폭);
    // 미사용 시 기본값 FOCAL 사용
    (void)calibrate;  // 미사용 경고 억제

    Mat frame;
    vector<Rect> faces;
    deque<double> hist;

    bool tooClose = false;
    auto t0 = steady_clock::now(), tBeep = t0;

    while (true)
    {
        cap >> frame;
        if (frame.empty()) break;

        cascade.detectMultiScale(frame, faces, 1.1, 5, 0, Size(80, 80));

        putMsg(frame, "POSTURE MONITOR", {10, 30}, {255, 255, 255}, 0.8);

        if (faces.empty())
        {
            putMsg(frame, "No Face Detected", {30, 70}, {128, 128, 128});
            tooClose = false;
            hist.clear();
        }
        else
        {
            auto& f = faces[0];
            rectangle(frame, f, {255, 0, 0}, 2);

            double dist = smoothDist(hist, FOCAL * FACE_W / f.width);

            putMsg(frame, "Dist: " + to_string((int)dist) + " cm",
                   {f.x, f.y - 10}, {255, 0, 0}, 0.7);

            judgePosture(frame, dist, tooClose, t0, tBeep);
        }

        faces.clear();
        imshow("Posture Detection", frame);
        if (waitKey(30) == 27) break;
    }

    return 0;
}
