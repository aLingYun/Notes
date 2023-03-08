## 准备素材
![](attachments/Pasted%20image%2020230308112913.png)
- `black.jpg` 是整个界面的背景；
- `rain.png` 是下落的流星雨，包括月亮；
- `star.png` 是满天的繁星；
- `love.jpg` 是月亮上的人影；
## 安装环境
```bash
yay -S gcc g++
yay -S sdl2 sdl2_ttf sdl2_net sdl2_mixer sdl2_image sdl2_gfx
```
本次的例子其实只用到了 `sdl2` 和 `sdl2_image` 这两个库。其他平台可以去 SDL 的官网下载对应的库文件。
## 代码
`Makefile` :
```Makefile
CC=g++
SRCS=main.cpp
CFLAGS=-lSDL2main -lSDL2 -lSDL2_image -lSDL2_mixer -lSDL2_net -lSDL2_ttf #-O2 #-mwindows

app: main.cpp
	$(CC) $(SRCS) $(CFLAGS) -o app
```
源文件：
```cpp
#include <SDL2/SDL.h>
#include <SDL2/SDL_blendmode.h>
#include <SDL2/SDL_events.h>
#include <SDL2/SDL_image.h>
#include <SDL2/SDL_render.h>
#include <SDL2/SDL_stdinc.h>
#include <SDL2/SDL_surface.h>
#include <SDL2/SDL_timer.h>
#include <SDL2/SDL_video.h>
#include <cstddef>
#include <cstdlib>
#include <iostream>

#define WINW 1920
#define WINH 1080
#define RAINNUM 2000

typedef struct {
  unsigned char r;
  unsigned char g;
  unsigned char b;
} Color;

typedef struct {
  int x;
  int y;
  int r;
  int speed;
  int alpha;
  Color rgb;
} Spot;

void drawSpot(Spot *spot, SDL_Renderer *rend, SDL_Texture *rain) {
  SDL_Rect rect = {spot->x, spot->y, spot->r, spot->r};
  SDL_SetTextureAlphaMod(rain, spot->alpha);
  SDL_SetTextureColorMod(rain, spot->rgb.r, spot->rgb.g, spot->rgb.b);
  SDL_RenderCopy(rend, rain, NULL, &rect);
}

void drawStar(SDL_Rect *rect, SDL_Renderer *rend, SDL_Texture *star) {
  SDL_RenderCopy(rend, star, NULL, rect);
}

void moveSpot(Spot *spot) {
  spot->y += spot->speed;
  spot->x += 1;
  if (spot->y > WINH) {
    spot->y = 0;
    spot->speed = rand() % 2 + 1;
  }
  if (spot->x > WINW) {
    spot->x = 0;
  }
}

int main(int argc, char *argv[]) {
  SDL_Init(SDL_INIT_VIDEO);
  SDL_Window *window = SDL_CreateWindow(
      "Rain Night", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED, WINW,
      WINH, SDL_WINDOW_FULLSCREEN_DESKTOP); // SDL_WINDOW_SHOWN);
  SDL_Renderer *rend = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);

  SDL_Surface *sur_rain = IMG_Load("rain.png");
  SDL_Surface *sur_black = IMG_Load("black.jpg");
  SDL_Surface *sur_star = IMG_Load("star.png");
  SDL_Surface *sur_love = IMG_Load("love.jpg");
  // SDL_SetColorKey(sur_love, SDL_TRUE, 0xFFFFFF);
  // SDL_SetColorKey(sur_love, SDL_TRUE, 0xEEEEEE);
  SDL_Texture *text_rain = SDL_CreateTextureFromSurface(rend, sur_rain);
  SDL_Texture *text_black = SDL_CreateTextureFromSurface(rend, sur_black);
  SDL_Texture *text_star = SDL_CreateTextureFromSurface(rend, sur_star);
  SDL_Texture *text_love = SDL_CreateTextureFromSurface(rend, sur_love);

  SDL_SetTextureBlendMode(text_black, SDL_BLENDMODE_BLEND);
  SDL_SetTextureAlphaMod(text_black, 20);
  SDL_SetTextureBlendMode(text_rain, SDL_BLENDMODE_BLEND);
  // SDL_SetTextureColorMod(text_rain, 166, 255, 255);

  SDL_Rect rect_moon = {-300, -100, 800, 800};
  SDL_Rect rect_window = {0, 0, WINW, WINH};
  SDL_Rect rect_love = {100, 200, 180, 250};

  Spot spots[RAINNUM];
  SDL_Rect rect_star[RAINNUM];
  for (int i = 0; i < RAINNUM; i++) {
    spots[i].x = rand() % WINW;
    spots[i].y = rand() % WINH;
    spots[i].r = rand() % 4 + 1;
    spots[i].speed = rand() % 2 + 1;
    spots[i].alpha = rand() % 255 + 1;
    spots[i].rgb.r = rand() % 255 + 1;
    spots[i].rgb.g = rand() % 255 + 1;
    spots[i].rgb.b = rand() % 255 + 1;
    rect_star[i].x = rand() % WINW;
    rect_star[i].y = rand() % WINH;
    rect_star[i].w = rect_star[i].h = rand() % 8 + 1;
  }

  SDL_Event event;
  bool quit = false;
  while (!quit) {
    while (SDL_PollEvent(&event)) {
      if (event.type == SDL_QUIT) {
        quit = true;
      }
    }
    SDL_RenderCopy(rend, text_black, NULL, &rect_window);
    SDL_SetTextureAlphaMod(text_rain, 255);
    SDL_SetTextureColorMod(text_rain, 255, 255, 255);
    SDL_RenderCopy(rend, text_rain, NULL, &rect_moon);
    // SDL_RenderCopy(rend, text_rain, NULL, &rect_rain);
    for (int i = 0; i < RAINNUM; i++) {
      drawStar(&rect_star[i], rend, text_star);
      drawSpot(&spots[i], rend, text_rain);
      moveSpot(&spots[i]);
    }
    SDL_RenderCopy(rend, text_love, NULL, &rect_love);
    SDL_RenderPresent(rend);
    SDL_Delay(15);
  }
  SDL_DestroyWindow(window);
  SDL_Quit();
  // std::cout << "Hello world" << std::endl;
  return 0;
}
```