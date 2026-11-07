#include <pspkernel.h>
#include <pspdisplay.h>
#include <pspctrl.h>
#include <pspgu.h>

PSP_MODULE_INFO("SMBPSP", 0, 1, 0);
PSP_MAIN_THREAD_ATTR(THREAD_ATTR_USER | THREAD_ATTR_VFPU);

#define BUF_WIDTH   512
#define SCR_WIDTH   480
#define SCR_HEIGHT  272
#define FRAME_SIZE  (BUF_WIDTH * SCR_HEIGHT * 4)

#define NES_W 256
#define NES_H 240

#define SCALE   ((float)SCR_HEIGHT / (float)NES_H)
#define OFFSET_X (((float)SCR_WIDTH - (float)NES_W * SCALE) / 2.0f)

static unsigned int __attribute__((aligned(16))) list[262144];

typedef struct {
    unsigned int color;
    float x, y, z;
} Vertex;

#define RGB(r, g, b) (0xFF000000u | ((unsigned)(b) << 16) | ((unsigned)(g) << 8) | (unsigned)(r))

static int exit_callback(int arg1, int arg2, void *common)
{
    (void)arg1; (void)arg2; (void)common;
    sceKernelExitGame();
    return 0;
}

static int callback_thread(SceSize args, void *argp)
{
    (void)args; (void)argp;
    int cbid = sceKernelCreateCallback("Exit Callback", exit_callback, NULL);
    sceKernelRegisterExitCallback(cbid);
    sceKernelSleepThreadCB();
    return 0;
}

static void setup_callbacks(void)
{
    int thid = sceKernelCreateThread("update_thread", callback_thread, 0x11, 0xFA0, 0, NULL);
    if (thid >= 0) {
        sceKernelStartThread(thid, 0, NULL);
    }
}

static void gfx_init(void)
{
    sceGuInit();
    sceGuStart(GU_DIRECT, list);

    sceGuDrawBuffer(GU_PSM_8888, (void *)0, BUF_WIDTH);
    sceGuDispBuffer(SCR_WIDTH, SCR_HEIGHT, (void *)FRAME_SIZE, BUF_WIDTH);
    sceGuDepthBuffer((void *)(FRAME_SIZE * 2), BUF_WIDTH);

    sceGuOffset(2048 - (SCR_WIDTH / 2), 2048 - (SCR_HEIGHT / 2));
    sceGuViewport(2048, 2048, SCR_WIDTH, SCR_HEIGHT);
    sceGuDepthRange(65535, 0);

    sceGuScissor(0, 0, SCR_WIDTH, SCR_HEIGHT);
    sceGuEnable(GU_SCISSOR_TEST);

    sceGuDisable(GU_DEPTH_TEST);
    sceGuDisable(GU_TEXTURE_2D);
    sceGuShadeModel(GU_FLAT);

    sceGuFinish();
    sceGuSync(0, 0);

    sceDisplayWaitVblankStart();
    sceGuDisplay(GU_TRUE);
}

static void draw_rect(float x, float y, float w, float h, unsigned int color)
{
    Vertex *v = (Vertex *)sceGuGetMemory(2 * sizeof(Vertex));

    v[0].color = color;
    v[0].x = OFFSET_X + x * SCALE;
    v[0].y = y * SCALE;
    v[0].z = 0.0f;

    v[1].color = color;
    v[1].x = OFFSET_X + (x + w) * SCALE;
    v[1].y = (y + h) * SCALE;
    v[1].z = 0.0f;

    sceGuDrawArray(GU_SPRITES,
                   GU_COLOR_8888 | GU_VERTEX_32BITF | GU_TRANSFORM_2D,
                   2, NULL, v);
}

#define GROUND_Y   208.0f
#define PLAYER_W   16.0f
#define PLAYER_H   16.0f

typedef struct {
    float x, y;
    float vx, vy;
    int on_ground;
} Player;

static void update(Player *p, const SceCtrlData *pad)
{
    const float speed   = 1.5f;
    const float gravity = 0.35f;
    const float jump_v  = -6.0f;

    p->vx = 0.0f;
    if (pad->Buttons & PSP_CTRL_LEFT)  p->vx = -speed;
    if (pad->Buttons & PSP_CTRL_RIGHT) p->vx =  speed;

    if ((pad->Buttons & PSP_CTRL_CROSS) && p->on_ground) {
        p->vy = jump_v;
        p->on_ground = 0;
    }

    p->vy += gravity;
    p->x += p->vx;
    p->y += p->vy;

    if (p->y + PLAYER_H >= GROUND_Y) {
        p->y = GROUND_Y - PLAYER_H;
        p->vy = 0.0f;
        p->on_ground = 1;
    }

    if (p->x < 0.0f) p->x = 0.0f;
    if (p->x > NES_W - PLAYER_W) p->x = NES_W - PLAYER_W;
}

static void render(const Player *p)
{
    sceGuStart(GU_DIRECT, list);

    sceGuClearColor(RGB(0, 0, 0));
    sceGuClear(GU_COLOR_BUFFER_BIT);

    draw_rect(0, 0, NES_W, GROUND_Y, RGB(92, 148, 252));
    draw_rect(0, GROUND_Y, NES_W, NES_H - GROUND_Y, RGB(200, 76, 12));
    draw_rect(p->x, p->y, PLAYER_W, PLAYER_H, RGB(220, 20, 20));

    sceGuFinish();
    sceGuSync(0, 0);
    sceDisplayWaitVblankStart();
    sceGuSwapBuffers();
}

int main(void)
{
    setup_callbacks();
    gfx_init();

    sceCtrlSetSamplingCycle(0);
    sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG);

    Player player = { 32.0f, GROUND_Y - PLAYER_H, 0.0f, 0.0f, 1 };
    SceCtrlData pad;

    int running = 1;
    while (running) {
        sceCtrlPeekBufferPositive(&pad, 1);

        if (pad.Buttons & PSP_CTRL_START) {
            running = 0;
        }

        update(&player, &pad);
        render(&player);
    }

    sceGuTerm();
    sceKernelExitGame();
    return 0;
}
