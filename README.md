step 1. Open visual studio 
step 2. create a console app project {name as u please} 
step 3. copy and paste the code as seen below 
step 4. run the debugger or run the exe
{additional instructions} how to get to the exe?
step 1. open ur project file in file explorer 
step 2. locate the "x64" file
step 3. after pressing the x64 file locate the dubug file
step 4. after clicking on the debug file locate the {project name} .exe



this is the code 
















#include <windows.h>
#include <string>

#define BLACK_COLOR RGB(20, 20, 20)
#define WHITE_COLOR RGB(255, 255, 255)
#define SPOTIFY_GREEN RGB(30, 215, 96)

const int radius = 20;
const int WIDTH = 300;
const int HEIGHT = 140;

enum AppState {
    WELCOME,
    ANIMATING_OUT,
    CONTROLS,
    ANIMATING_IN
};

AppState currentState = WELCOME;
int alpha = 255;  // Current window alpha (opacity)

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam);

void SendMediaKey(WORD vk) {
    keybd_event(vk, 0, 0, 0);
}

void StartFadeOut(HWND hwnd) {
    currentState = ANIMATING_OUT;
    alpha = 255;
    SetTimer(hwnd, 1, 15, NULL);
}

void StartFadeIn(HWND hwnd) {
    currentState = ANIMATING_IN;
    alpha = 0;
    SetTimer(hwnd, 1, 15, NULL);
}

// Add to Windows startup by writing to registry
void AddToStartup(const wchar_t* exePath) {
    HKEY hKey;
    // Open the Run key in Current User hive
    if (RegOpenKeyEx(HKEY_CURRENT_USER,
        L"Software\\Microsoft\\Windows\\CurrentVersion\\Run",
        0, KEY_READ | KEY_WRITE, &hKey) == ERROR_SUCCESS) {

        // Check if value already exists
        DWORD dataSize = 0;
        LONG queryResult = RegQueryValueEx(hKey, L"SpotifyHotkeys", NULL, NULL, NULL, &dataSize);
        if (queryResult != ERROR_SUCCESS) {
            // Not found, add it
            RegSetValueEx(hKey, L"SpotifyHotkeys", 0, REG_SZ,
                (const BYTE*)exePath,
                (DWORD)((wcslen(exePath) + 1) * sizeof(wchar_t)));
        }

        RegCloseKey(hKey);
    }
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int nCmdShow) {
    // Add to startup on first run
    wchar_t exePath[MAX_PATH];
    GetModuleFileName(NULL, exePath, MAX_PATH);
    AddToStartup(exePath);

    const wchar_t CLASS_NAME[] = L"SpotifyHotkeysRounded";

    WNDCLASS wc = {};
    wc.lpfnWndProc = WindowProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = CLASS_NAME;
    wc.hbrBackground = NULL;

    RegisterClass(&wc);

    int screenWidth = GetSystemMetrics(SM_CXSCREEN);
    int screenHeight = GetSystemMetrics(SM_CYSCREEN);
    int x = (screenWidth - WIDTH) / 2;
    int y = (screenHeight - HEIGHT) / 2;

    HWND hwnd = CreateWindowEx(
        WS_EX_TOPMOST | WS_EX_LAYERED | WS_EX_TOOLWINDOW,
        CLASS_NAME,
        L"Spotify Hotkeys",
        WS_POPUP | WS_VISIBLE,
        x, y, WIDTH, HEIGHT,
        NULL, NULL, hInstance, NULL
    );

    if (!hwnd) {
        MessageBox(NULL, L"Failed to create window.", L"Error", MB_ICONERROR);
        return 0;
    }

    HRGN hRgn = CreateRoundRectRgn(0, 0, WIDTH + 1, HEIGHT + 1, radius, radius);
    SetWindowRgn(hwnd, hRgn, TRUE);

    SetLayeredWindowAttributes(hwnd, 0, 255, LWA_ALPHA);

    RegisterHotKey(hwnd, 1, 0, VK_F8);
    RegisterHotKey(hwnd, 2, 0, VK_F9);
    RegisterHotKey(hwnd, 3, 0, VK_F10);
    RegisterHotKey(hwnd, 4, 0, VK_F7);  // F7 to start
    RegisterHotKey(hwnd, 5, 0, VK_F4);  // F4 to shutdown

    MSG msg = {};
    while (GetMessage(&msg, NULL, 0, 0)) {
        if (msg.message == WM_HOTKEY) {
            if (msg.wParam == 4 && currentState == WELCOME) {
                StartFadeOut(msg.hwnd);
            }
            else if (msg.wParam == 5) {
                DestroyWindow(msg.hwnd);
            }
            else if (currentState == CONTROLS) {
                switch (msg.wParam) {
                case 1: SendMediaKey(VK_MEDIA_PREV_TRACK); break;
                case 2: SendMediaKey(VK_MEDIA_PLAY_PAUSE); break;
                case 3: SendMediaKey(VK_MEDIA_NEXT_TRACK); break;
                }
            }
        }
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return 0;
}

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    switch (uMsg) {
    case WM_PAINT: {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(hwnd, &ps);

        RECT clientRect;
        GetClientRect(hwnd, &clientRect);

        HBRUSH blackBrush = CreateSolidBrush(BLACK_COLOR);
        FillRect(hdc, &clientRect, blackBrush);

        SetBkMode(hdc, TRANSPARENT);

        HFONT font = CreateFont(20, 0, 0, 0, FW_BOLD, FALSE, FALSE, FALSE,
            ANSI_CHARSET, OUT_DEFAULT_PRECIS, CLIP_DEFAULT_PRECIS, DEFAULT_QUALITY,
            DEFAULT_PITCH | FF_SWISS, TEXT("Segoe UI"));
        SelectObject(hdc, font);

        if (currentState == WELCOME || currentState == ANIMATING_OUT) {
            SetTextColor(hdc, WHITE_COLOR);
            const wchar_t* welcomeLines[] = {
                L"Welcome to Spotify Hotkeys",
                L"Press [F7] to begin",
                L"On 60% keyboards, press Fn + 7 for F7"
            };

            for (int i = 0; i < 2; ++i) {
                SIZE textSize;
                GetTextExtentPoint32(hdc, welcomeLines[i], (int)wcslen(welcomeLines[i]), &textSize);
                int xPos = (clientRect.right - clientRect.left - textSize.cx) / 2;
                int yPos = (i == 0) ? 30 : 70;
                TextOut(hdc, xPos, yPos, welcomeLines[i], wcslen(welcomeLines[i]));
            }

            // Smaller instruction line for 60% keyboards
            HFONT fontSmall = CreateFont(14, 0, 0, 0, FW_NORMAL, FALSE, FALSE, FALSE,
                ANSI_CHARSET, OUT_DEFAULT_PRECIS, CLIP_DEFAULT_PRECIS, DEFAULT_QUALITY,
                DEFAULT_PITCH | FF_SWISS, TEXT("Segoe UI"));
            SelectObject(hdc, fontSmall);
            SIZE textSize;
            GetTextExtentPoint32(hdc, welcomeLines[2], (int)wcslen(welcomeLines[2]), &textSize);
            int xPos = (clientRect.right - clientRect.left - textSize.cx) / 2;
            int yPos = 100;
            TextOut(hdc, xPos, yPos, welcomeLines[2], wcslen(welcomeLines[2]));
            DeleteObject(fontSmall);
        }
        else if (currentState == CONTROLS || currentState == ANIMATING_IN) {
            SetTextColor(hdc, WHITE_COLOR);
            const wchar_t* leftText[] = {
                L"[F8]   Previous Track",
                L"[F9]   Play / Pause",
                L"[F10]  Next Track"
            };
            const wchar_t* rightText = L"[F4]   Exit";

            int y = 15;
            for (int i = 0; i < 3; ++i) {
                TextOut(hdc, 15, y, leftText[i], wcslen(leftText[i]));
                y += 30;
            }

            // Right-align the exit text on the same vertical line as the first hotkey line
            SIZE rightTextSize;
            GetTextExtentPoint32(hdc, rightText, (int)wcslen(rightText), &rightTextSize);
            int rightX = clientRect.right - rightTextSize.cx - 15;
            TextOut(hdc, rightX, 15, rightText, wcslen(rightText));
        }

        DeleteObject(font);
        DeleteObject(blackBrush);

        EndPaint(hwnd, &ps);
        return 0;
    }
    case WM_TIMER: {
        if (wParam == 1) {
            if (currentState == ANIMATING_OUT) {
                alpha -= 15;
                if (alpha <= 0) {
                    alpha = 0;
                    currentState = CONTROLS;
                    StartFadeIn(hwnd);
                }
                SetLayeredWindowAttributes(hwnd, 0, (BYTE)alpha, LWA_ALPHA);
            }
            else if (currentState == ANIMATING_IN) {
                alpha += 15;
                if (alpha >= 255) {
                    alpha = 255;
                    currentState = CONTROLS;
                    KillTimer(hwnd, 1);
                }
                SetLayeredWindowAttributes(hwnd, 0, (BYTE)alpha, LWA_ALPHA);
            }
            InvalidateRect(hwnd, NULL, TRUE);
        }
        return 0;
    }
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    case WM_NCHITTEST: {
        LRESULT hit = DefWindowProc(hwnd, uMsg, wParam, lParam);
        if (hit == HTCLIENT) return HTCAPTION;
        return hit;
    }
    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
