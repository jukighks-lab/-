개발일지
=======

## 형식
    1. 프로젝트 개요

      9~10 이하동일

    2. 프로젝트 구조 & 리소스

      9~10 이하동일

    3. 개발 내용

    기존의 단순 거리 측정 기능을 개선하여 사용자의 자세를 분석할 수 있는 기능을 추가하였다. 
    얼굴 크기를 이용해 사용자와 카메라 사이의 거리를 계산하고, 측정값의 오차를 줄이기 위해 최근 거리 데이터의 평균값을 사용하는 기능을 구현하였다. 
    또한 사용자가 바른 자세를 취한 상태에서 기준 거리를 설정할 수 있도록 하여 개인별 사용 환경에 맞는 자세 판별이 가능하도록 개선하였다.
    기준 거리와 현재 거리를 비교하여 정상, 주의, 거북목 의심의 3단계로 자세를 분류하도록 구현하였으며, 일시적인 움직임에 의한 오탐지를 방지하기 위해 위험 자세가 3초 이상 지속될 경우에만 경고가 표시되도록 하였다. 
    추가로 화면에 자세 상태를 실시간으로 출력하고, 거북목 의심 상태가 유지될 경우 경고음을 발생시키는 기능을 개발하였다.


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
const double FOCAL = 700.0, FACE_W = 15.0;
const int    SMOOTH = 5;

// 절대적인 거리 대신, 기준 자세에서부터 얼마나 앞으로 나왔는지를 판별하는 마진(여백) cm
const double WARN_MARGIN = 5.0;   // 5cm 앞으로 오면 경고 시작
const double DANGER_MARGIN = 10.0; // 10cm 앞으로 오면 위험 판정 (거북목)

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

// ===== 자세 판정 + 경보 (기준 거리 baselineDist 추가) =====
void judgePosture(Mat& frame, double dist, double baselineDist, bool& tooClose,
    steady_clock::time_point& t0,
    steady_clock::time_point& tBeep)
{
    auto now = steady_clock::now();
    double el = duration_cast<seconds>(now - t0).count();
    double beepEl = duration_cast<seconds>(now - tBeep).count();

    // 기준 거리에서 설정한 마진을 뺀 값을 경계선으로 사용
    double warnDist = baselineDist - WARN_MARGIN;
    double dangerDist = baselineDist - DANGER_MARGIN;

    if (dist >= warnDist)
    {
        putMsg(frame, "GOOD POSTURE", { 30, 70 }, { 0, 255, 0 });
        tooClose = false;
    }
    else if (dist >= dangerDist)
    {
        putMsg(frame, "TOO CLOSE", { 30,  70 }, { 0, 255, 255 });
        putMsg(frame, "Please Keep Distance", { 30, 110 }, { 0, 255, 255 }, 0.8);
        tooClose = false;
    }
    else
    {
        if (!tooClose) { tooClose = true; t0 = tBeep = now; el = 0; }

        if (el < 3)
            putMsg(frame, "Warning in " + to_string(3 - (int)el) + "s",
                { 30, 70 }, { 0, 255, 255 });
        else
        {
            putMsg(frame, "FORWARD HEAD POSTURE!", { 30,  70 }, { 0, 0, 255 });
            putMsg(frame, "Please Sit Further Back", { 30, 110 }, { 0, 0, 255 }, 0.8);
            if (beepEl >= 5) { Beep(1000, 500); tBeep = now; }  // 5초마다 반복 경보
        }
    }
}

// ===== main =====
int main()
{
    utils::logging::setLogLevel(utils::logging::LOG_LEVEL_ERROR);

    CascadeClassifier cascade;
    VideoCapture cap(0, CAP_DSHOW);

    if (!cascade.load("haarcascade_frontalface_default.xml") || !cap.isOpened())
    {
        cout << "Error: Model not found or camera cannot be opened." << endl;
        return -1;
    }

    cap.set(CAP_PROP_FRAME_WIDTH, 640);
    cap.set(CAP_PROP_FRAME_HEIGHT, 480);

    Mat frame;
    vector<Rect> faces;
    deque<double> hist;

    bool tooClose = false;
    auto t0 = steady_clock::now(), tBeep = t0;

    // 초기 기준 자세 관련 변수
    bool isBaselineSet = false;
    double baselineDist = 0.0;

    while (true)
    {
        cap >> frame;
        if (frame.empty()) break;

        cascade.detectMultiScale(frame, faces, 1.1, 5, 0, Size(80, 80));

        putMsg(frame, "POSTURE MONITOR", { 10, 30 }, { 255, 255, 255 }, 0.8);

        if (faces.empty())
        {
            putMsg(frame, "No Face Detected", { 30, 70 }, { 128, 128, 128 });
            tooClose = false;
            hist.clear();
        }
        else
        {
            auto& f = faces[0];
            rectangle(frame, f, { 255, 0, 0 }, 2);

            double dist = smoothDist(hist, FOCAL * FACE_W / f.width);

            putMsg(frame, "Dist: " + to_string((int)dist) + " cm",
                { f.x, f.y - 10 }, { 255, 0, 0 }, 0.7);

            // 1. 기준점이 설정되지 않았을 때 안내
            if (!isBaselineSet)
            {
                putMsg(frame, "Sit straight and press 'S' to set baseline", { 15, 450 }, { 0, 165, 255 }, 0.7);
                putMsg(frame, "WAITING FOR CALIBRATION...", { 30, 70 }, { 0, 165, 255 }, 0.8);
            }
            // 2. 기준점이 설정된 이후 자세 모니터링
            else
            {
                putMsg(frame, "Baseline: " + to_string((int)baselineDist) + " cm", { 15, 450 }, { 255, 255, 255 }, 0.6);
                judgePosture(frame, dist, baselineDist, tooClose, t0, tBeep);
            }
        }

        faces.clear();
        imshow("Posture Detection", frame);

        int key = waitKey(30);

        // ESC 키 누르면 종료
        if (key == 27) break;

        // 's' 또는 'S' 키를 누르면 현재 거리를 기준 자세(Baseline)로 설정
        if ((key == 's' || key == 'S') && !hist.empty())
        {
            baselineDist = smoothDist(hist, hist.back()); // 현재 저장된 거리들로 평균 측정
            isBaselineSet = true;
            cout << "Baseline set to: " << baselineDist << " cm" << endl;
        }
    }

    return 0;
}
