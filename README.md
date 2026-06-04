#include <windows.h>
#include <GL/glut.h>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <cstdlib>
#include <ctime>
#include <vector>
#include <string>

// ======================= CONSTANTS =======================
const int   SW = 1280, SH = 720;
const float PI = 3.14159265f;
const float GRAVITY = -20.0f, JUMP_V = 7.0f;
const float PWALK = 5.0f, PRUN = 9.0f;         // units per SECOND
const float ZWALK = 1.2f, ZCHASE = 3.0f;        // units per SECOND
const float ZATKRNG = 2.2f, ZDETRNG = 22.0f;
const float BULSPD = 80.0f;                     // units per SECOND
const int   MAXHP = 100, ZDMG = 7;
const int   GUNDMG = 35, KNIFEDMG = 90, RIFLEDMG = 55;
const int   ZSCORE = 10;
const float CHALF = 60.0f;
const float MOUSE_SENS = 0.002f;

enum GameState {MENU, PLAYING, PAUSED, GAMEOVER};
enum CamMode   {FIRST, THIRD, FREECAM};
enum WeapType  {W_GUN, W_KNIFE, W_RIFLE};
enum FurnType  {F_TABLE, F_CHAIR, F_BED, F_CLOSET, F_SHELF, F_SWITCH};

// ======================= FORWARD DECLARATIONS =======================
struct Vec3;
struct AABB;
struct Door;
struct Furniture;
struct Room;
struct Building;
struct Bullet;
struct Zombie;

void spawnZombie();
bool checkCollision(Vec3 pos, float radius);
void shoot();
void initWorld();
void checkInteractions();
void updateGame();

// ======================= Vec3 =======================
struct Vec3 {
    float x, y, z;
    Vec3() : x(0), y(0), z(0) {}
    Vec3(float a, float b, float c) : x(a), y(b), z(c) {}
    Vec3 operator+(Vec3 v) const { return Vec3(x+v.x, y+v.y, z+v.z); }
    Vec3 operator-(Vec3 v) const { return Vec3(x-v.x, y-v.y, z-v.z); }
    Vec3 operator*(float s) const { return Vec3(x*s, y*s, z*s); }
    Vec3 operator/(float s) const { return Vec3(x/s, y/s, z/s); }
    float len() const { return sqrtf(x*x + y*y + z*z); }
    float lenXZ() const { return sqrtf(x*x + z*z); }
    Vec3 norm() const { float l = len(); return l > 0 ? Vec3(x/l, y/l, z/l) : Vec3(0,0,0); }
    float dist(Vec3 v) const { return (*this - v).len(); }
    float distXZ(Vec3 v) const { float dx = x-v.x, dz = z-v.z; return sqrtf(dx*dx + dz*dz); }
};

// ======================= AABB =======================
struct AABB {
    Vec3 mn, mx;
    AABB() : mn(Vec3(0,0,0)), mx(Vec3(0,0,0)) {}
    AABB(Vec3 a, Vec3 b) : mn(a), mx(b) {}
    bool insideXZ(Vec3 p) const { return p.x >= mn.x && p.x <= mx.x && p.z >= mn.z && p.z <= mx.z; }
    bool collides(AABB o) const {
        return mn.x < o.mx.x && mx.x > o.mn.x &&
               mn.y < o.mx.y && mx.y > o.mn.y &&
               mn.z < o.mx.z && mx.z > o.mn.z;
    }
    Vec3 center() const { return Vec3((mn.x+mx.x)/2.0f, (mn.y+mx.y)/2.0f, (mn.z+mx.z)/2.0f); }
};

// ======================= Door =======================
struct Door {
    Vec3 hinge;
    float width, height;
    float angle, target;
    bool open;
    int side;
    AABB closedBox;
    Door() : angle(0), target(0), open(false), side(0), width(2), height(3) {}
    void toggle() { open = !open; target = open ? 90.0f : 0.0f; }
    void update(float dt) {
        float spd = 120.0f * dt;
        if (angle < target) angle += spd;
        if (angle > target) angle -= spd;
        if (fabsf(angle - target) < spd) angle = target;
    }
    AABB currentBox() {
        if (open) return AABB(Vec3(-999,-999,-999), Vec3(-999,-999,-999));
        return closedBox;
    }
};

// ======================= Furniture =======================
struct Furniture {
    FurnType type;
    Vec3 pos;
    float rot;
    AABB box;
    float colR, colG, colB;
    Furniture() : type(F_TABLE), rot(0), colR(0.5f), colG(0.3f), colB(0.1f) {}
};

// ======================= Room =======================
struct Room {
    Vec3 pos;
    float w, d, h;
    bool lightOn;
    Vec3 lightPos;
    int glLightID;
    std::vector<Furniture> furn;
    std::vector<AABB> walls;
    int switchIdx;
    Room() : lightOn(false), glLightID(GL_LIGHT2), switchIdx(-1) {}
};

// ======================= Building =======================
struct Building {
    Vec3 pos;
    float w, d, h;
    float colR, colG, colB;
    bool enterable;
    std::vector<Door> doors;
    std::vector<Room> rooms;
    std::vector<AABB> outerWalls;
    std::vector<AABB> allColliders() {
        std::vector<AABB> c = outerWalls;
        for (size_t i = 0; i < rooms.size(); i++) {
            for (size_t j = 0; j < rooms[i].walls.size(); j++)
                c.push_back(rooms[i].walls[j]);
            for (size_t j = 0; j < rooms[i].furn.size(); j++)
                c.push_back(rooms[i].furn[j].box);
        }
        for (size_t i = 0; i < doors.size(); i++) {
            AABB db = doors[i].currentBox();
            if (db.mn.x > -900) c.push_back(db);
        }
        return c;
    }
};

// ======================= Bullet =======================
struct Bullet {
    Vec3 pos, dir;
    float life;
    bool active;
    Bullet(Vec3 p, Vec3 d) : pos(p), dir(d), life(2.0f), active(true) {}
    void update(float dt) {
        pos = pos + dir * (BULSPD * dt);
        life -= dt;
        if (life <= 0) active = false;
    }
};

// ======================= Zombie =======================
struct Zombie {
    Vec3 pos;
    float yaw;
    float hp;
    float walkPhase;
    bool chasing;
    bool attacking;
    float atkTimer;
    bool alive;
    float dieTimer;
    Zombie(Vec3 p) : pos(p), yaw((float)(rand()%360)), hp(100), walkPhase(0),
                     chasing(false), attacking(false), atkTimer(0),
                     alive(true), dieTimer(0) {}
    void update(Vec3 playerPos, float dt);
};

// ======================= GLOBALS =======================
GameState gState = MENU;
CamMode   gCam   = FIRST;
WeapType  gWeap  = W_GUN;

Vec3  pPos(0, 0, 5);
float pYaw = 0, pPitch = 0;
float pVY = 0;
bool  pOnGround = true;
int   pHP = MAXHP;
int   pScore = 0;
int   pLives = 3;
bool  flashlightOn = true;
int   pAmmo[3] = {60, 999, 40};
float pShootCD = 0;
float pSurvTime = 0;
int   pLevel = 1;

float camDist = 5.0f;
float camYaw = 0, camPitch = 20.0f;

bool keys[256] = {};
bool skeys[256] = {};   // special keys

std::vector<Building> buildings;
std::vector<AABB>     wallColliders;
std::vector<Zombie>   zombies;
std::vector<Bullet>   bullets;
std::vector<Vec3>     treePositions;
std::vector<Vec3>     streetLights;
std::vector<Vec3>     carPositions;
std::vector<float>    carRots;
std::vector<Vec3>     carColors;

float dayTime = 0.3f;
bool  dayNightOn = true;
int   menuSel = 0;
std::string interactText = "";
Door*   nearDoor = NULL;
Room*   nearSwitch = NULL;
bool    showInstructions = false;
float   bonusTimer = 0;
float   lastTime = 0;

// ======================= HELPERS =======================
inline float fmax2(float a, float b) { return a > b ? a : b; }
inline float fmin2(float a, float b) { return a < b ? a : b; }

bool isShiftHeld() {
    return (GetAsyncKeyState(VK_SHIFT) & 0x8000) != 0;
}

// ======================= DRAWING HELPERS =======================
void drawBox(float sx, float sy, float sz) {
    glPushMatrix();
    glScalef(sx, sy, sz);
    glBegin(GL_QUADS);
    glNormal3f(0,0,1);
    glVertex3f(-0.5f,-0.5f, 0.5f); glVertex3f( 0.5f,-0.5f, 0.5f);
    glVertex3f( 0.5f, 0.5f, 0.5f); glVertex3f(-0.5f, 0.5f, 0.5f);
    glNormal3f(0,0,-1);
    glVertex3f( 0.5f,-0.5f,-0.5f); glVertex3f(-0.5f,-0.5f,-0.5f);
    glVertex3f(-0.5f, 0.5f,-0.5f); glVertex3f( 0.5f, 0.5f,-0.5f);
    glNormal3f(-1,0,0);
    glVertex3f(-0.5f,-0.5f,-0.5f); glVertex3f(-0.5f,-0.5f, 0.5f);
    glVertex3f(-0.5f, 0.5f, 0.5f); glVertex3f(-0.5f, 0.5f,-0.5f);
    glNormal3f(1,0,0);
    glVertex3f( 0.5f,-0.5f, 0.5f); glVertex3f( 0.5f,-0.5f,-0.5f);
    glVertex3f( 0.5f, 0.5f,-0.5f); glVertex3f( 0.5f, 0.5f, 0.5f);
    glNormal3f(0,1,0);
    glVertex3f(-0.5f, 0.5f, 0.5f); glVertex3f( 0.5f, 0.5f, 0.5f);
    glVertex3f( 0.5f, 0.5f,-0.5f); glVertex3f(-0.5f, 0.5f,-0.5f);
    glNormal3f(0,-1,0);
    glVertex3f(-0.5f,-0.5f,-0.5f); glVertex3f( 0.5f,-0.5f,-0.5f);
    glVertex3f( 0.5f,-0.5f, 0.5f); glVertex3f(-0.5f,-0.5f, 0.5f);
    glEnd();
    glPopMatrix();
}

void drawCylinder(float r, float h, int sl) {
    glBegin(GL_QUAD_STRIP);
    for (int i = 0; i <= sl; i++) {
        float a = 2.0f * PI * i / sl;
        float nx = cosf(a), nz = sinf(a);
        glNormal3f(nx, 0, nz);
        glVertex3f(r*nx, h, r*nz); glVertex3f(r*nx, 0, r*nz);
    }
    glEnd();
    glBegin(GL_TRIANGLE_FAN); glNormal3f(0,1,0); glVertex3f(0,h,0);
    for (int i = 0; i <= sl; i++) { float a = 2.0f*PI*i/sl; glVertex3f(r*cosf(a), h, r*sinf(a)); }
    glEnd();
    glBegin(GL_TRIANGLE_FAN); glNormal3f(0,-1,0); glVertex3f(0,0,0);
    for (int i = sl; i >= 0; i--) { float a = 2.0f*PI*i/sl; glVertex3f(r*cosf(a), 0, r*sinf(a)); }
    glEnd();
}

void setColor(float r, float g, float b, float a = 1.0f) {
    GLfloat d[4] = {r, g, b, a};
    GLfloat s[4] = {0.2f, 0.2f, 0.2f, 1.0f};
    glMaterialfv(GL_FRONT_AND_BACK, GL_AMBIENT_AND_DIFFUSE, d);
    glMaterialfv(GL_FRONT_AND_BACK, GL_SPECULAR, s);
    glMaterialf(GL_FRONT_AND_BACK, GL_SHININESS, 10);
    glColor4f(r, g, b, a);
}

// ======================= HUMANOID =======================
void drawHumanoid(float r, float g, float b, float walkP, bool attacking, float scale) {
    glPushMatrix();
    glScalef(scale, scale, scale);
    float ls = sinf(walkP) * 0.4f;
    float as2 = attacking ? 0.8f : sinf(walkP) * 0.3f;

    setColor(r*0.8f, g*0.8f, b*0.8f);
    glPushMatrix(); glTranslatef(0, 1.0f, 0); drawBox(0.4f, 0.6f, 0.25f); glPopMatrix();

    setColor(r, g, b);
    glPushMatrix(); glTranslatef(0, 1.55f, 0); glutSolidSphere(0.2f, 12, 12);
    setColor(1, 0.2f, 0.2f);
    glPushMatrix(); glTranslatef(-0.07f, 0.05f, -0.18f); glutSolidSphere(0.04f, 6, 6); glPopMatrix();
    glPushMatrix(); glTranslatef( 0.07f, 0.05f, -0.18f); glutSolidSphere(0.04f, 6, 6); glPopMatrix();
    glPopMatrix();

    setColor(r*0.7f, g*0.7f, b*0.7f);
    glPushMatrix(); glTranslatef(-0.3f, 1.2f, 0); glRotatef(as2*30, 1, 0, 0);
    drawBox(0.12f, 0.55f, 0.12f); glPopMatrix();
    glPushMatrix(); glTranslatef(0.3f, 1.2f, 0);
    if (attacking) glRotatef(-60, 1, 0, 0); else glRotatef(-as2*30, 1, 0, 0);
    drawBox(0.12f, 0.55f, 0.12f); glPopMatrix();

    setColor(r*0.5f, g*0.5f, b*0.5f);
    glPushMatrix(); glTranslatef(-0.1f, 0.45f, 0); glRotatef( ls*30, 1, 0, 0);
    drawBox(0.14f, 0.6f, 0.14f); glPopMatrix();
    glPushMatrix(); glTranslatef( 0.1f, 0.45f, 0); glRotatef(-ls*30, 1, 0, 0);
    drawBox(0.14f, 0.6f, 0.14f); glPopMatrix();

    glPopMatrix();
}

// ======================= ZOMBIE UPDATE =======================
void Zombie::update(Vec3 playerPos, float dt) {
    if (!alive) { dieTimer += dt; return; }
    float d = pos.distXZ(playerPos);
    chasing  = (d < ZDETRNG);
    attacking = (d < ZATKRNG);

    if (chasing) {
        float ddx = playerPos.x - pos.x, ddz = playerPos.z - pos.z;
        yaw = atan2f(ddx, -ddz) * 180.0f / PI;
        float spd = ZCHASE * dt;
        Vec3 fwd(sinf(yaw * PI / 180.0f) * spd, 0, -cosf(yaw * PI / 180.0f) * spd);
        Vec3 np = pos + fwd;
        bool blocked = false;
        AABB zBox(Vec3(np.x-0.3f, 0, np.z-0.3f), Vec3(np.x+0.3f, 1.8f, np.z+0.3f));
        for (size_t i = 0; i < wallColliders.size(); i++) {
            if (zBox.collides(wallColliders[i])) { blocked = true; break; }
        }
        if (!blocked && fabsf(np.x) < CHALF && fabsf(np.z) < CHALF) pos = np;
    } else {
        yaw += ((rand() % 100) - 50) * 0.02f;
        float spd = ZWALK * dt;
        Vec3 fwd(sinf(yaw * PI / 180.0f) * spd, 0, -cosf(yaw * PI / 180.0f) * spd);
        Vec3 np = pos + fwd;
        if (fabsf(np.x) < CHALF && fabsf(np.z) < CHALF) {
            bool blocked = false;
            AABB zBox(Vec3(np.x-0.3f, 0, np.z-0.3f), Vec3(np.x+0.3f, 1.8f, np.z+0.3f));
            for (size_t i = 0; i < wallColliders.size(); i++) {
                if (zBox.collides(wallColliders[i])) { blocked = true; break; }
            }
            if (!blocked) pos = np; else yaw += 90;
        } else { yaw += 90; }
    }
    walkPhase += dt * (chasing ? 8.0f : 4.0f);
    if (attacking) atkTimer += dt; else atkTimer = 0;
}

// ======================= BUILDING SETUP =======================
void addSimpleBuilding(std::vector<Building>& bv, float x, float z, float w, float d, float h,
                       float r, float g, float b) {
    Building bld;
    bld.pos = Vec3(x, 0, z); bld.w = w; bld.d = d; bld.h = h;
    bld.colR = r; bld.colG = g; bld.colB = b; bld.enterable = false;
    float t = 0.3f;
    bld.outerWalls.push_back(AABB(Vec3(x-w/2, 0, z-d/2),   Vec3(x+w/2, h, z-d/2+t)));
    bld.outerWalls.push_back(AABB(Vec3(x-w/2, 0, z+d/2-t), Vec3(x+w/2, h, z+d/2)));
    bld.outerWalls.push_back(AABB(Vec3(x-w/2, 0, z-d/2),   Vec3(x-w/2+t, h, z+d/2)));
    bld.outerWalls.push_back(AABB(Vec3(x+w/2-t, 0, z-d/2), Vec3(x+w/2, h, z+d/2)));
    bv.push_back(bld);
}

void setupBuilding1(Building& b) {
    b.pos = Vec3(-18, 0, -18); b.w = 14; b.d = 12; b.h = 8;
    b.colR = 0.6f; b.colG = 0.55f; b.colB = 0.5f; b.enterable = true;
    float t = 0.3f;
    float bx = b.pos.x, bz = b.pos.z, bw = b.w, bd = b.d, bh = b.h;
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx+bw/2, bh, bz-bd/2+t)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz+bd/2-t), Vec3(bx+bw/2, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx-bw/2+t, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx+bw/2-t, 0, bz-bd/2), Vec3(bx+bw/2, bh, bz-bd/2+3)));
    b.outerWalls.push_back(AABB(Vec3(bx+bw/2-t, 0, bz-bd/2+5), Vec3(bx+bw/2, bh, bz+bd/2)));
    Door d; d.hinge = Vec3(bx+bw/2-t/2, 0, bz-bd/2+3);
    d.width = 2; d.height = 3; d.angle = 0; d.target = 0; d.open = false; d.side = 0;
    d.closedBox = AABB(Vec3(bx+bw/2-t, 0, bz-bd/2+3), Vec3(bx+bw/2, bh, bz-bd/2+5));
    b.doors.push_back(d);
    float iwX = bx;
    b.outerWalls.push_back(AABB(Vec3(iwX-t/2, 0, bz-t/2), Vec3(iwX+t/2, bh, bz-3)));
    b.outerWalls.push_back(AABB(Vec3(iwX-t/2, 0, bz+3),   Vec3(iwX+t/2, bh, bz+bd/2-t)));
    Door d2; d2.hinge = Vec3(iwX, 0, bz-3);
    d2.width = 2; d2.height = 3; d2.angle = 0; d2.target = 0; d2.open = false; d2.side = 0;
    d2.closedBox = AABB(Vec3(iwX-t/2, 0, bz-3), Vec3(iwX+t/2, bh, bz+3));
    b.doors.push_back(d2);

    Room r1; r1.pos = Vec3(bx, 0, bz-bd/4); r1.w = bw-t*2; r1.d = bd/2-t; r1.h = bh;
    r1.lightOn = false; r1.lightPos = Vec3(bx, bh-0.5f, bz-bd/4);
    r1.glLightID = GL_LIGHT2; r1.switchIdx = 0;
    Furniture f;
    f.type = F_TABLE; f.pos = Vec3(bx-3, 0.5f, bz-bd/4); f.rot = 0;
    f.colR = 0.6f; f.colG = 0.4f; f.colB = 0.2f;
    f.box = AABB(Vec3(f.pos.x-1, 0, f.pos.z-0.6f), Vec3(f.pos.x+1, 1, f.pos.z+0.6f));
    r1.furn.push_back(f);
    f.type = F_CHAIR; f.pos = Vec3(bx-3, 0.3f, bz-bd/4+1.2f); f.rot = 0;
    f.colR = 0.5f; f.colG = 0.3f; f.colB = 0.15f;
    f.box = AABB(Vec3(f.pos.x-0.3f, 0, f.pos.z-0.3f), Vec3(f.pos.x+0.3f, 0.8f, f.pos.z+0.3f));
    r1.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx+bw/2-t-0.2f, 1.2f, bz-bd/2+t+0.5f); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.1f, f.pos.y-0.1f, f.pos.z-0.02f),
                 Vec3(f.pos.x+0.1f, f.pos.y+0.1f, f.pos.z+0.02f));
    r1.furn.push_back(f); r1.switchIdx = 2;

    Room r2; r2.pos = Vec3(bx, 0, bz+bd/4); r2.w = bw-t*2; r2.d = bd/2-t; r2.h = bh;
    r2.lightOn = false; r2.lightPos = Vec3(bx, bh-0.5f, bz+bd/4);
    r2.glLightID = GL_LIGHT3; r2.switchIdx = 0;
    f.type = F_BED; f.pos = Vec3(bx+3, 0.4f, bz+bd/4); f.rot = 0;
    f.colR = 0.3f; f.colG = 0.3f; f.colB = 0.7f;
    f.box = AABB(Vec3(f.pos.x-0.8f, 0, f.pos.z-1.2f), Vec3(f.pos.x+0.8f, 0.8f, f.pos.z+1.2f));
    r2.furn.push_back(f);
    f.type = F_CLOSET; f.pos = Vec3(bx-bw/2+t+0.6f, 1.0f, bz+bd/4); f.rot = 0;
    f.colR = 0.4f; f.colG = 0.25f; f.colB = 0.1f;
    f.box = AABB(Vec3(f.pos.x-0.4f, 0, f.pos.z-0.5f), Vec3(f.pos.x+0.4f, 2, f.pos.z+0.5f));
    r2.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx+bw/2-t-0.2f, 1.2f, bz+t+0.5f); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.1f, f.pos.y-0.1f, f.pos.z-0.02f),
                 Vec3(f.pos.x+0.1f, f.pos.y+0.1f, f.pos.z+0.02f));
    r2.furn.push_back(f); r2.switchIdx = 2;
    b.rooms.push_back(r1); b.rooms.push_back(r2);
}

void setupBuilding2(Building& b) {
    b.pos = Vec3(18, 0, -18); b.w = 10; b.d = 10; b.h = 6;
    b.colR = 0.7f; b.colG = 0.65f; b.colB = 0.55f; b.enterable = true;
    float t = 0.3f;
    float bx = b.pos.x, bz = b.pos.z, bw = b.w, bd = b.d, bh = b.h;
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx+bw/2, bh, bz-bd/2+t)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz+bd/2-t), Vec3(bx-2, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx+2, 0, bz+bd/2-t),    Vec3(bx+bw/2, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx-bw/2+t, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx+bw/2-t, 0, bz-bd/2), Vec3(bx+bw/2, bh, bz+bd/2)));
    Door d; d.hinge = Vec3(bx-2, 0, bz+bd/2-t/2);
    d.width = 2; d.height = 3; d.angle = 0; d.target = 0; d.open = false; d.side = 0;
    d.closedBox = AABB(Vec3(bx-2, 0, bz+bd/2-t), Vec3(bx+2, bh, bz+bd/2));
    b.doors.push_back(d);
    Room r1; r1.pos = b.pos; r1.w = bw-t*2; r1.d = bd-t*2; r1.h = bh;
    r1.lightOn = false; r1.lightPos = Vec3(bx, bh-0.5f, bz);
    r1.glLightID = GL_LIGHT4; r1.switchIdx = 0;
    Furniture f;
    f.type = F_TABLE; f.pos = Vec3(bx, 0.5f, bz); f.rot = 0;
    f.colR = 0.55f; f.colG = 0.35f; f.colB = 0.15f;
    f.box = AABB(Vec3(f.pos.x-0.8f, 0, f.pos.z-0.5f), Vec3(f.pos.x+0.8f, 1, f.pos.z+0.5f));
    r1.furn.push_back(f);
    f.type = F_SHELF; f.pos = Vec3(bx-bw/2+t+0.3f, 1.0f, bz-2); f.rot = 0;
    f.colR = 0.5f; f.colG = 0.3f; f.colB = 0.1f;
    f.box = AABB(Vec3(f.pos.x-0.2f, 0, f.pos.z-0.8f), Vec3(f.pos.x+0.2f, 2, f.pos.z+0.8f));
    r1.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx-2+0.2f, 1.2f, bz+bd/2-t-0.2f); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.1f, f.pos.y-0.1f, f.pos.z-0.02f),
                 Vec3(f.pos.x+0.1f, f.pos.y+0.1f, f.pos.z+0.02f));
    r1.furn.push_back(f); r1.switchIdx = 2;
    b.rooms.push_back(r1);
}

void setupBuilding3(Building& b) {
    b.pos = Vec3(-18, 0, 18); b.w = 8; b.d = 10; b.h = 5;
    b.colR = 0.65f; b.colG = 0.6f; b.colB = 0.5f; b.enterable = true;
    float t = 0.3f;
    float bx = b.pos.x, bz = b.pos.z, bw = b.w, bd = b.d, bh = b.h;
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx+bw/2, bh, bz-bd/2+t)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz+bd/2-t), Vec3(bx+bw/2, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2),   Vec3(bx-bw/2+t, bh, bz-2)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz+2),      Vec3(bx-bw/2+t, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx+bw/2-t, 0, bz-bd/2), Vec3(bx+bw/2, bh, bz+bd/2)));
    Door d; d.hinge = Vec3(bx-bw/2+t/2, 0, bz-2);
    d.width = 2; d.height = 3; d.angle = 0; d.target = 0; d.open = false; d.side = 1;
    d.closedBox = AABB(Vec3(bx-bw/2, 0, bz-2), Vec3(bx-bw/2+t, bh, bz+2));
    b.doors.push_back(d);
    Room r1; r1.pos = b.pos; r1.w = bw-t*2; r1.d = bd-t*2; r1.h = bh;
    r1.lightOn = false; r1.lightPos = Vec3(bx, bh-0.5f, bz);
    r1.glLightID = GL_LIGHT5; r1.switchIdx = 0;
    Furniture f;
    f.type = F_BED; f.pos = Vec3(bx+2, 0.4f, bz+2); f.rot = 0;
    f.colR = 0.4f; f.colG = 0.2f; f.colB = 0.2f;
    f.box = AABB(Vec3(f.pos.x-0.7f, 0, f.pos.z-1.1f), Vec3(f.pos.x+0.7f, 0.8f, f.pos.z+1.1f));
    r1.furn.push_back(f);
    f.type = F_CHAIR; f.pos = Vec3(bx-2, 0.3f, bz-2); f.rot = 0;
    f.colR = 0.5f; f.colG = 0.3f; f.colB = 0.1f;
    f.box = AABB(Vec3(f.pos.x-0.3f, 0, f.pos.z-0.3f), Vec3(f.pos.x+0.3f, 0.8f, f.pos.z+0.3f));
    r1.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx-bw/2+t+0.2f, 1.2f, bz-1); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.02f, f.pos.y-0.1f, f.pos.z-0.1f),
                 Vec3(f.pos.x+0.02f, f.pos.y+0.1f, f.pos.z+0.1f));
    r1.furn.push_back(f); r1.switchIdx = 2;
    b.rooms.push_back(r1);
}

void setupBuilding4(Building& b) {
    b.pos = Vec3(18, 0, 18); b.w = 12; b.d = 12; b.h = 7;
    b.colR = 0.55f; b.colG = 0.55f; b.colB = 0.58f; b.enterable = true;
    float t = 0.3f;
    float bx = b.pos.x, bz = b.pos.z, bw = b.w, bd = b.d, bh = b.h;
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2), Vec3(bx-2, bh, bz-bd/2+t)));
    b.outerWalls.push_back(AABB(Vec3(bx+2, 0, bz-bd/2),    Vec3(bx+bw/2, bh, bz-bd/2+t)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz+bd/2-t), Vec3(bx+bw/2, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx-bw/2, 0, bz-bd/2), Vec3(bx-bw/2+t, bh, bz+bd/2)));
    b.outerWalls.push_back(AABB(Vec3(bx+bw/2-t, 0, bz-bd/2), Vec3(bx+bw/2, bh, bz+bd/2)));
    Door d; d.hinge = Vec3(bx-2, 0, bz-bd/2+t/2);
    d.width = 2; d.height = 3; d.angle = 0; d.target = 0; d.open = false; d.side = 0;
    d.closedBox = AABB(Vec3(bx-2, 0, bz-bd/2), Vec3(bx+2, bh, bz-bd/2+t));
    b.doors.push_back(d);
    b.outerWalls.push_back(AABB(Vec3(bx-t/2, 0, bz-2), Vec3(bx+t/2, bh, bz+bd/2-t)));
    Door d2; d2.hinge = Vec3(bx, 0, bz-2);
    d2.width = 2; d2.height = 3; d2.angle = 0; d2.target = 0; d2.open = false; d2.side = 0;
    d2.closedBox = AABB(Vec3(bx-t/2, 0, bz-2), Vec3(bx+t/2, bh, bz));
    b.doors.push_back(d2);

    Room r1; r1.pos = Vec3(bx, 0, bz-bd/4); r1.w = bw-t*2; r1.d = bd/2-t; r1.h = bh;
    r1.lightOn = false; r1.lightPos = Vec3(bx, bh-0.5f, bz-bd/4);
    r1.glLightID = GL_LIGHT6; r1.switchIdx = 0;
    Furniture f;
    f.type = F_TABLE; f.pos = Vec3(bx+3, 0.5f, bz-bd/4); f.rot = 0;
    f.colR = 0.6f; f.colG = 0.4f; f.colB = 0.2f;
    f.box = AABB(Vec3(f.pos.x-0.8f, 0, f.pos.z-0.5f), Vec3(f.pos.x+0.8f, 1, f.pos.z+0.5f));
    r1.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx-1.5f, 1.2f, bz-bd/2+t+0.2f); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.1f, f.pos.y-0.1f, f.pos.z-0.02f),
                 Vec3(f.pos.x+0.1f, f.pos.y+0.1f, f.pos.z+0.02f));
    r1.furn.push_back(f); r1.switchIdx = 1;

    Room r2; r2.pos = Vec3(bx, 0, bz+bd/4); r2.w = bw-t*2; r2.d = bd/2-t; r2.h = bh;
    r2.lightOn = false; r2.lightPos = Vec3(bx, bh-0.5f, bz+bd/4);
    r2.glLightID = GL_LIGHT7; r2.switchIdx = 0;
    f.type = F_BED; f.pos = Vec3(bx-3, 0.4f, bz+bd/4); f.rot = 0;
    f.colR = 0.3f; f.colG = 0.4f; f.colB = 0.6f;
    f.box = AABB(Vec3(f.pos.x-0.7f, 0, f.pos.z-1.1f), Vec3(f.pos.x+0.7f, 0.8f, f.pos.z+1.1f));
    r2.furn.push_back(f);
    f.type = F_CLOSET; f.pos = Vec3(bx+bw/2-t-0.5f, 1.0f, bz+bd/4); f.rot = 0;
    f.colR = 0.45f; f.colG = 0.3f; f.colB = 0.15f;
    f.box = AABB(Vec3(f.pos.x-0.4f, 0, f.pos.z-0.5f), Vec3(f.pos.x+0.4f, 2, f.pos.z+0.5f));
    r2.furn.push_back(f);
    f.type = F_SWITCH; f.pos = Vec3(bx-t-0.2f, 1.2f, bz); f.rot = 0;
    f.colR = 0.9f; f.colG = 0.9f; f.colB = 0.8f;
    f.box = AABB(Vec3(f.pos.x-0.1f, f.pos.y-0.1f, f.pos.z-0.02f),
                 Vec3(f.pos.x+0.1f, f.pos.y+0.1f, f.pos.z+0.02f));
    r2.furn.push_back(f); r2.switchIdx = 2;
    b.rooms.push_back(r1); b.rooms.push_back(r2);
}

// ======================= SPAWN ZOMBIE =======================
void spawnZombie() {
    for (int tries = 0; tries < 50; tries++) {
        float zx = -CHALF + 5 + (float)(rand() % 110);
        float zz = -CHALF + 5 + (float)(rand() % 110);
        if (Vec3(zx, 0, zz).distXZ(pPos) > 20) {
            bool inWall = false;
            for (size_t j = 0; j < wallColliders.size(); j++) {
                if (wallColliders[j].insideXZ(Vec3(zx, 0, zz))) { inWall = true; break; }
            }
            if (!inWall) { zombies.push_back(Zombie(Vec3(zx, 0, zz))); return; }
        }
    }
    zombies.push_back(Zombie(Vec3(pPos.x + 30, 0, pPos.z + 30)));
}

// ======================= INIT WORLD =======================
void initWorld() {
    srand((unsigned)time(NULL));
    buildings.clear(); zombies.clear(); bullets.clear();
    treePositions.clear(); streetLights.clear();
    carPositions.clear(); carRots.clear(); carColors.clear();
    wallColliders.clear();

    Building b1, b2, b3, b4;
    setupBuilding1(b1); buildings.push_back(b1);
    setupBuilding2(b2); buildings.push_back(b2);
    setupBuilding3(b3); buildings.push_back(b3);
    setupBuilding4(b4); buildings.push_back(b4);

    addSimpleBuilding(buildings, -40, 0, 8, 8, 5, 0.7f, 0.6f, 0.5f);
    addSimpleBuilding(buildings,  40, 0, 10, 8, 6, 0.6f, 0.65f, 0.55f);
    addSimpleBuilding(buildings, -40,-25, 6, 6, 4, 0.65f, 0.55f, 0.5f);
    addSimpleBuilding(buildings,  40,-25, 8, 6, 5, 0.55f, 0.6f, 0.65f);
    addSimpleBuilding(buildings,   0,-40, 10, 6, 4, 0.7f, 0.55f, 0.45f);
    addSimpleBuilding(buildings,   0, 40, 8, 8, 5, 0.6f, 0.6f, 0.6f);
    addSimpleBuilding(buildings, -35, 30, 6, 6, 4.5f, 0.5f, 0.65f, 0.55f);
    addSimpleBuilding(buildings,  35,-35, 7, 7, 5, 0.6f, 0.5f, 0.55f);

    for (size_t i = 0; i < buildings.size(); i++) {
        std::vector<AABB> c2 = buildings[i].allColliders();
        wallColliders.insert(wallColliders.end(), c2.begin(), c2.end());
    }

    for (int i = 0; i < 25; i++) {
        float tx = -CHALF + 5 + (float)(rand() % 110);
        float tz = -CHALF + 5 + (float)(rand() % 110);
        bool ok = true;
        for (size_t j = 0; j < wallColliders.size(); j++) {
            Vec3 c3 = wallColliders[j].center();
            if (Vec3(tx, 0, tz).distXZ(c3) < 4) { ok = false; break; }
        }
        if (ok) treePositions.push_back(Vec3(tx, 0, tz));
    }

    for (float z = -CHALF + 5; z < CHALF; z += 12) {
        streetLights.push_back(Vec3(-3.5f, 0, z));
        streetLights.push_back(Vec3( 3.5f, 0, z));
    }
    for (float x = -CHALF + 5; x < CHALF; x += 12) {
        if (fabsf(x) < 5) continue;
        streetLights.push_back(Vec3(x, 0, -3.5f));
        streetLights.push_back(Vec3(x, 0,  3.5f));
    }

    carPositions.push_back(Vec3(-1, 0,-30)); carRots.push_back(0);   carColors.push_back(Vec3(0.8f,0.1f,0.1f));
    carPositions.push_back(Vec3( 1, 0, 20)); carRots.push_back(180); carColors.push_back(Vec3(0.1f,0.1f,0.8f));
    carPositions.push_back(Vec3(-25,0, -1)); carRots.push_back(90);  carColors.push_back(Vec3(0.9f,0.9f,0.1f));
    carPositions.push_back(Vec3( 30,0,  1)); carRots.push_back(270); carColors.push_back(Vec3(0.1f,0.8f,0.1f));
    carPositions.push_back(Vec3(-1, 0, 38)); carRots.push_back(0);   carColors.push_back(Vec3(0.7f,0.7f,0.7f));

    for (size_t i = 0; i < carPositions.size(); i++) {
        float cx = carPositions[i].x, cz = carPositions[i].z;
        wallColliders.push_back(AABB(Vec3(cx-1.2f, 0, cz-2.5f), Vec3(cx+1.2f, 1.5f, cz+2.5f)));
    }

    wallColliders.push_back(AABB(Vec3(-CHALF, 0,-CHALF), Vec3(CHALF, 4,-CHALF+0.5f)));
    wallColliders.push_back(AABB(Vec3(-CHALF, 0, CHALF-0.5f), Vec3(CHALF, 4, CHALF)));
    wallColliders.push_back(AABB(Vec3(-CHALF, 0,-CHALF), Vec3(-CHALF+0.5f, 4, CHALF)));
    wallColliders.push_back(AABB(Vec3(CHALF-0.5f, 0,-CHALF), Vec3(CHALF, 4, CHALF)));

    int nz = 5 + pLevel * 3;
    for (int i = 0; i < nz; i++) spawnZombie();
}

// ======================= COLLISION =======================
bool checkCollision(Vec3 pos, float radius) {
    AABB pBox(Vec3(pos.x-radius, pos.y, pos.z-radius),
              Vec3(pos.x+radius, pos.y+1.8f, pos.z+radius));
    for (size_t i = 0; i < wallColliders.size(); i++) {
        if (pBox.collides(wallColliders[i])) return true;
    }
    return false;
}

// ======================= INTERACTIONS =======================
void checkInteractions() {
    interactText = ""; nearDoor = NULL; nearSwitch = NULL;
    for (size_t bi = 0; bi < buildings.size(); bi++) {
        Building& bld = buildings[bi];
        if (!bld.enterable) continue;
        for (size_t di = 0; di < bld.doors.size(); di++) {
            Door& d = bld.doors[di];
            Vec3 dc = d.hinge + Vec3(d.width/2, d.height/2, 0);
            if (pPos.dist(dc) < 3.5f) {
                nearDoor = &d;
                interactText = d.open ? "Press E to Close Door" : "Press E to Open Door";
                return;
            }
        }
        for (size_t ri = 0; ri < bld.rooms.size(); ri++) {
            Room& r = bld.rooms[ri];
            if (r.switchIdx >= 0 && r.switchIdx < (int)r.furn.size()) {
                Furniture& sw = r.furn[r.switchIdx];
                if (pPos.dist(sw.pos) < 3.0f) {
                    nearSwitch = &r;
                    interactText = r.lightOn ? "Press E: Light Off" : "Press E: Light On";
                    return;
                }
            }
        }
    }
}

// ======================= SHOOTING =======================
void shoot() {
    if (pShootCD > 0) return;
    Vec3 dir(sinf(pYaw)*cosf(pPitch), sinf(pPitch), -cosf(pYaw)*cosf(pPitch));
    Vec3 bpos = pPos + Vec3(0, 1.3f, 0) + dir * 0.5f;
    switch (gWeap) {
        case W_GUN:
            if (pAmmo[0] <= 0) return;
            pAmmo[0]--; pShootCD = 0.25f;
            bullets.push_back(Bullet(bpos, dir));
            break;
        case W_KNIFE:
            pShootCD = 0.5f;
            for (size_t i = 0; i < zombies.size(); i++) {
                if (!zombies[i].alive) continue;
                if (pPos.dist(zombies[i].pos) < 3.0f) {
                    Vec3 toZ = (zombies[i].pos - pPos).norm();
                    float dot = dir.x*toZ.x + dir.z*toZ.z;
                    if (dot > 0.3f) {
                        zombies[i].hp -= KNIFEDMG;
                        if (zombies[i].hp <= 0) { zombies[i].alive = false; pScore += ZSCORE; }
                    }
                }
            }
            break;
        case W_RIFLE:
            if (pAmmo[2] <= 0) return;
            pAmmo[2]--; pShootCD = 0.12f;
            bullets.push_back(Bullet(bpos, dir));
            break;
    }
}

// ======================= RENDERING =======================
void drawGround() {
    setColor(0.25f, 0.45f, 0.15f);
    glBegin(GL_QUADS); glNormal3f(0,1,0);
    glVertex3f(-CHALF, 0,-CHALF); glVertex3f(CHALF, 0,-CHALF);
    glVertex3f(CHALF, 0, CHALF);  glVertex3f(-CHALF, 0, CHALF); glEnd();
    setColor(0.3f, 0.3f, 0.3f);
    glBegin(GL_QUADS); glNormal3f(0,1,0);
    glVertex3f(-3,-0.01f,-CHALF); glVertex3f(3,-0.01f,-CHALF);
    glVertex3f(3,-0.01f, CHALF);  glVertex3f(-3,-0.01f, CHALF); glEnd();
    glBegin(GL_QUADS); glNormal3f(0,1,0);
    glVertex3f(-CHALF,-0.01f,-3); glVertex3f(CHALF,-0.01f,-3);
    glVertex3f(CHALF,-0.01f, 3);  glVertex3f(-CHALF,-0.01f, 3); glEnd();
    setColor(0.9f, 0.9f, 0.2f);
    for (float z = -CHALF; z < CHALF; z += 4) {
        glBegin(GL_QUADS); glNormal3f(0,1,0);
        glVertex3f(-0.1f,-0.005f, z); glVertex3f( 0.1f,-0.005f, z);
        glVertex3f( 0.1f,-0.005f, z+2); glVertex3f(-0.1f,-0.005f, z+2); glEnd();
    }
    for (float x = -CHALF; x < CHALF; x += 4) {
        glBegin(GL_QUADS); glNormal3f(0,1,0);
        glVertex3f(x,-0.005f,-0.1f); glVertex3f(x+2,-0.005f,-0.1f);
        glVertex3f(x+2,-0.005f, 0.1f); glVertex3f(x,-0.005f, 0.1f); glEnd();
    }
    setColor(0.6f, 0.58f, 0.55f);
    glBegin(GL_QUADS); glNormal3f(0,1,0);
    glVertex3f(-4.5f, 0.05f,-CHALF); glVertex3f(-3, 0.05f,-CHALF);
    glVertex3f(-3, 0.05f, CHALF); glVertex3f(-4.5f, 0.05f, CHALF); glEnd();
    glBegin(GL_QUADS); glNormal3f(0,1,0);
    glVertex3f(3, 0.05f,-CHALF); glVertex3f(4.5f, 0.05f,-CHALF);
    glVertex3f(4.5f, 0.05f, CHALF); glVertex3f(3, 0.05f, CHALF); glEnd();
}

void drawTree(Vec3 p) {
    glPushMatrix(); glTranslatef(p.x, 0, p.z);
    setColor(0.4f, 0.25f, 0.1f);
    glPushMatrix(); glTranslatef(0, 1.5f, 0); glRotatef(-90, 1, 0, 0);
    drawCylinder(0.25f, 3, 8); glPopMatrix();
    setColor(0.15f, 0.5f, 0.1f);
    glPushMatrix(); glTranslatef(0, 4, 0); glutSolidSphere(1.5f, 10, 10); glPopMatrix();
    setColor(0.1f, 0.45f, 0.05f);
    glPushMatrix(); glTranslatef(0.5f, 4.5f, 0.3f); glutSolidSphere(1.0f, 8, 8); glPopMatrix();
    glPopMatrix();
}

void drawStreetLight(Vec3 p) {
    glPushMatrix(); glTranslatef(p.x, 0, p.z);
    setColor(0.3f, 0.3f, 0.3f);
    glPushMatrix(); glTranslatef(0, 3, 0); glRotatef(-90, 1, 0, 0);
    drawCylinder(0.08f, 6, 6); glPopMatrix();
    glPushMatrix(); glTranslatef(0, 5.5f, 0); drawBox(1.0f, 0.06f, 0.06f); glPopMatrix();
    setColor(1, 1, 0.8f);
    glPushMatrix(); glTranslatef(0.5f, 5.3f, 0); glutSolidSphere(0.12f, 8, 8); glPopMatrix();
    glPopMatrix();
}

void drawCar(Vec3 p, float rot, Vec3 col) {
    glPushMatrix(); glTranslatef(p.x, 0, p.z); glRotatef(rot, 0, 1, 0);
    setColor(col.x, col.y, col.z);
    glPushMatrix(); glTranslatef(0, 0.5f, 0); drawBox(2.0f, 0.8f, 4.5f); glPopMatrix();
    setColor(col.x*0.8f, col.y*0.8f, col.z*0.8f);
    glPushMatrix(); glTranslatef(0, 1.1f,-0.3f); drawBox(1.7f, 0.6f, 2.5f); glPopMatrix();
    setColor(0.15f, 0.15f, 0.15f);
    glPushMatrix(); glTranslatef(-1.0f, 0.25f, 1.3f); glutSolidSphere(0.3f, 8, 8); glPopMatrix();
    glPushMatrix(); glTranslatef( 1.0f, 0.25f, 1.3f); glutSolidSphere(0.3f, 8, 8); glPopMatrix();
    glPushMatrix(); glTranslatef(-1.0f, 0.25f,-1.3f); glutSolidSphere(0.3f, 8, 8); glPopMatrix();
    glPushMatrix(); glTranslatef( 1.0f, 0.25f,-1.3f); glutSolidSphere(0.3f, 8, 8); glPopMatrix();
    setColor(0.5f, 0.7f, 0.9f);
    glPushMatrix(); glTranslatef(0, 1.2f,-1.6f); drawBox(1.5f, 0.4f, 0.05f); glPopMatrix();
    glPopMatrix();
}

void drawSkybox() {
    float s = CHALF + 10;
    glDisable(GL_LIGHTING);
    float sunAngle = dayTime * 2 * PI;
    float sunY = sinf(sunAngle);
    float skyR = 0.2f + 0.3f * fmax2(0.0f, sunY);
    float skyG = 0.3f + 0.3f * fmax2(0.0f, sunY);
    float skyB = 0.6f + 0.3f * fmax2(0.0f, sunY);
    if (sunY < 0) { skyR *= 0.3f; skyG *= 0.3f; skyB *= 0.4f; }
    glBegin(GL_QUADS);
    glColor3f(skyR*0.8f, skyG*0.8f, skyB);
    glVertex3f(-s, s,-s); glVertex3f(s, s,-s); glVertex3f(s, s, s); glVertex3f(-s, s, s);
    glColor3f(0.3f, 0.25f, 0.2f);
    glVertex3f(-s,-0.1f,-s); glVertex3f(s,-0.1f,-s); glVertex3f(s,-0.1f, s); glVertex3f(-s,-0.1f, s);
    glEnd();
    glEnable(GL_LIGHTING);
}

void drawDoorMesh(Door& d) {
    glPushMatrix(); glTranslatef(d.hinge.x, d.hinge.y, d.hinge.z);
    if (d.side == 0) glRotatef(d.angle, 0, 1, 0); else glRotatef(-d.angle, 0, 1, 0);
    setColor(0.45f, 0.3f, 0.15f);
    float hw = d.width / 2;
    if (d.side == 0) {
        glPushMatrix(); glTranslatef(hw, d.height/2, 0); drawBox(d.width, d.height, 0.1f); glPopMatrix();
    } else {
        glPushMatrix(); glTranslatef(-hw, d.height/2, 0); drawBox(d.width, d.height, 0.1f); glPopMatrix();
    }
    setColor(0.8f, 0.7f, 0.2f);
    if (d.side == 0) {
        glPushMatrix(); glTranslatef(0.2f, d.height/2, 0.08f); glutSolidSphere(0.06f, 6, 6); glPopMatrix();
    } else {
        glPushMatrix(); glTranslatef(-0.2f, d.height/2, 0.08f); glutSolidSphere(0.06f, 6, 6); glPopMatrix();
    }
    glPopMatrix();
}

void drawFurniture(Furniture& f) {
    glPushMatrix(); glTranslatef(f.pos.x, f.pos.y, f.pos.z); glRotatef(f.rot, 0, 1, 0);
    setColor(f.colR, f.colG, f.colB);
    switch (f.type) {
        case F_TABLE:
            glPushMatrix(); glTranslatef(0, 0.35f, 0); drawBox(1.6f, 0.08f, 1.0f); glPopMatrix();
            glPushMatrix(); glTranslatef(-0.7f, 0.15f,-0.4f); drawBox(0.06f, 0.3f, 0.06f); glPopMatrix();
            glPushMatrix(); glTranslatef( 0.7f, 0.15f,-0.4f); drawBox(0.06f, 0.3f, 0.06f); glPopMatrix();
            glPushMatrix(); glTranslatef(-0.7f, 0.15f, 0.4f); drawBox(0.06f, 0.3f, 0.06f); glPopMatrix();
            glPushMatrix(); glTranslatef( 0.7f, 0.15f, 0.4f); drawBox(0.06f, 0.3f, 0.06f); glPopMatrix();
            break;
        case F_CHAIR:
            glPushMatrix(); glTranslatef(0, 0.2f, 0); drawBox(0.5f, 0.05f, 0.5f); glPopMatrix();
            glPushMatrix(); glTranslatef(0, 0.5f,-0.22f); drawBox(0.5f, 0.5f, 0.05f); glPopMatrix();
            break;
        case F_BED:
            glPushMatrix(); drawBox(1.4f, 0.35f, 2.2f); glPopMatrix();
            setColor(0.9f, 0.9f, 0.85f);
            glPushMatrix(); glTranslatef(0, 0.25f,-0.8f); drawBox(0.8f, 0.12f, 0.4f); glPopMatrix();
            setColor(f.colR*0.7f, f.colG*0.7f, f.colB*0.7f);
            glPushMatrix(); glTranslatef(0, 0.45f,-1.05f); drawBox(1.4f, 0.5f, 0.08f); glPopMatrix();
            break;
        case F_CLOSET:
            glPushMatrix(); drawBox(0.8f, 1.8f, 0.9f); glPopMatrix();
            setColor(0.7f, 0.6f, 0.2f);
            glPushMatrix(); glTranslatef(-0.1f, 1.0f, 0.46f); glutSolidSphere(0.04f, 6, 6); glPopMatrix();
            glPushMatrix(); glTranslatef( 0.1f, 1.0f, 0.46f); glutSolidSphere(0.04f, 6, 6); glPopMatrix();
            break;
        case F_SHELF:
            for (int i = 0; i < 4; i++) {
                glPushMatrix(); glTranslatef(0, 0.3f + i*0.4f, 0); drawBox(0.3f, 0.04f, 1.4f); glPopMatrix();
            }
            break;
        case F_SWITCH:
            glPushMatrix(); drawBox(0.1f, 0.15f, 0.04f); glPopMatrix();
            setColor(1, 1, 0);
            glPushMatrix(); glTranslatef(0, 0.03f, 0.025f); drawBox(0.06f, 0.06f, 0.01f); glPopMatrix();
            break;
    }
    glPopMatrix();
}

void drawBuilding(Building& b) {
    setColor(b.colR, b.colG, b.colB);
    glPushMatrix(); glTranslatef(b.pos.x, b.h/2, b.pos.z); drawBox(b.w, b.h, b.d); glPopMatrix();
    setColor(0.4f, 0.2f, 0.1f);
    glPushMatrix(); glTranslatef(b.pos.x, b.h+0.15f, b.pos.z); drawBox(b.w+0.4f, 0.3f, b.d+0.4f); glPopMatrix();
    setColor(0.6f, 0.75f, 0.85f);
    for (float wx = b.pos.x - b.w/2 + 2; wx < b.pos.x + b.w/2 - 1; wx += 3) {
        for (float wy = 2; wy < b.h - 1; wy += 2.5f) {
            glPushMatrix(); glTranslatef(wx, wy, b.pos.z - b.d/2 - 0.02f); drawBox(1.0f, 1.2f, 0.05f); glPopMatrix();
            glPushMatrix(); glTranslatef(wx, wy, b.pos.z + b.d/2 + 0.02f); drawBox(1.0f, 1.2f, 0.05f); glPopMatrix();
        }
    }
    if (b.enterable) {
        setColor(0.5f, 0.4f, 0.3f);
        glPushMatrix(); glTranslatef(b.pos.x, 0.02f, b.pos.z); drawBox(b.w-0.6f, 0.04f, b.d-0.6f); glPopMatrix();
        for (size_t i = 0; i < b.rooms.size(); i++) {
            Room& r = b.rooms[i];
            setColor(0.85f, 0.85f, 0.8f);
            glPushMatrix(); glTranslatef(r.pos.x, b.h-0.05f, r.pos.z); drawBox(r.w, 0.1f, r.d); glPopMatrix();
            if (r.lightOn) setColor(1, 1, 0.85f); else setColor(0.3f, 0.3f, 0.3f);
            glPushMatrix(); glTranslatef(r.lightPos.x, r.lightPos.y, r.lightPos.z);
            glutSolidSphere(r.lightOn ? 0.2f : 0.15f, 8, 8); glPopMatrix();
            for (size_t j = 0; j < r.furn.size(); j++) drawFurniture(r.furn[j]);
        }
        for (size_t i = 0; i < b.doors.size(); i++) drawDoorMesh(b.doors[i]);
        for (size_t i = 5; i < b.outerWalls.size(); i++) {
            AABB& wb = b.outerWalls[i];
            Vec3 c3 = wb.center();
            float sx = wb.mx.x - wb.mn.x, sy = wb.mx.y - wb.mn.y, sz = wb.mx.z - wb.mn.z;
            setColor(0.75f, 0.7f, 0.65f);
            glPushMatrix(); glTranslatef(c3.x, c3.y, c3.z); drawBox(sx, sy, sz); glPopMatrix();
        }
    }
}

void drawZombie(Zombie& z) {
    if (!z.alive) {
        float t2 = fmin2(z.dieTimer, 1.0f);
        glPushMatrix(); glTranslatef(z.pos.x, t2*0.5f, z.pos.z);
        glRotatef(z.yaw, 0, 1, 0); glRotatef(t2*80, 1, 0, 0);
        drawHumanoid(0.3f, 0.5f, 0.2f, 0, false, 1.0f); glPopMatrix();
        return;
    }
    glPushMatrix(); glTranslatef(z.pos.x, 0, z.pos.z); glRotatef(z.yaw, 0, 1, 0);
    drawHumanoid(0.3f, 0.5f, 0.2f, z.walkPhase, z.attacking, 1.0f);
    setColor(0.6f, 0, 0);
    glPushMatrix(); glTranslatef(0.3f, 1.0f,-0.2f); glutSolidSphere(0.1f, 6, 6); glPopMatrix();
    glPopMatrix();
}

void drawWeaponFP() {
    if (gCam != FIRST) return;
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);

    switch (gWeap) {
        case W_GUN:
            glColor3f(0.2f, 0.2f, 0.2f);
            glBegin(GL_QUADS); glVertex2f(SW-200, 80); glVertex2f(SW-80, 80);
            glVertex2f(SW-80, 160); glVertex2f(SW-200, 160); glEnd();
            glColor3f(0.15f, 0.15f, 0.15f);
            glBegin(GL_QUADS); glVertex2f(SW-160, 160); glVertex2f(SW-120, 160);
            glVertex2f(SW-130, 260); glVertex2f(SW-150, 260); glEnd();
            if (pShootCD > 0.1f) {
                glColor3f(1, 0.8f, 0);
                glBegin(GL_TRIANGLES); glVertex2f(SW-140, 260); glVertex2f(SW-120, 300);
                glVertex2f(SW-160, 300); glEnd();
            }
            break;
        case W_KNIFE:
            glColor3f(0.5f, 0.5f, 0.5f);
            glBegin(GL_QUADS); glVertex2f(SW-160, 100); glVertex2f(SW-120, 100);
            glVertex2f(SW-110, 250); glVertex2f(SW-130, 250); glEnd();
            glColor3f(0.4f, 0.25f, 0.1f);
            glBegin(GL_QUADS); glVertex2f(SW-165, 60); glVertex2f(SW-115, 60);
            glVertex2f(SW-120, 110); glVertex2f(SW-160, 110); glEnd();
            break;
        case W_RIFLE:
            glColor3f(0.25f, 0.2f, 0.2f);
            glBegin(GL_QUADS); glVertex2f(SW-220, 90); glVertex2f(SW-60, 90);
            glVertex2f(SW-60, 140); glVertex2f(SW-220, 140); glEnd();
            glColor3f(0.4f, 0.25f, 0.1f);
            glBegin(GL_QUADS); glVertex2f(SW-220, 80); glVertex2f(SW-180, 80);
            glVertex2f(SW-180, 150); glVertex2f(SW-220, 150); glEnd();
            glColor3f(0.15f, 0.15f, 0.15f);
            glBegin(GL_QUADS); glVertex2f(SW-85, 130); glVertex2f(SW-65, 130);
            glVertex2f(SW-70, 280); glVertex2f(SW-80, 280); glEnd();
            if (pShootCD > 0.08f) {
                glColor3f(1, 0.6f, 0);
                glBegin(GL_TRIANGLES); glVertex2f(SW-75, 280); glVertex2f(SW-55, 320);
                glVertex2f(SW-95, 320); glEnd();
            }
            break;
    }

    glColor3f(0, 1, 0);
    glBegin(GL_LINES);
    glVertex2f(SW/2-10, SH/2); glVertex2f(SW/2+10, SH/2);
    glVertex2f(SW/2, SH/2-10); glVertex2f(SW/2, SH/2+10);
    glEnd();

    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix();
    glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

// ======================= HUD =======================
void drawText2D(int x, int y, const char* txt, float r, float g, float b) {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glColor3f(r, g, b); glRasterPos2f(x, y);
    for (const char* c = txt; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, *c);
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

void drawText2Dsmall(int x, int y, const char* txt, float r, float g, float b) {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glColor3f(r, g, b); glRasterPos2f(x, y);
    for (const char* c = txt; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

void drawHUD() {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);

    glColor3f(0.3f, 0.1f, 0.1f);
    glBegin(GL_QUADS); glVertex2f(20, SH-40); glVertex2f(220, SH-40);
    glVertex2f(220, SH-20); glVertex2f(20, SH-20); glEnd();
    float hp = (float)pHP / (float)MAXHP;
    glColor3f(1-hp, hp, 0);
    glBegin(GL_QUADS); glVertex2f(20, SH-40); glVertex2f(20+200*hp, SH-40);
    glVertex2f(20+200*hp, SH-20); glVertex2f(20, SH-20); glEnd();
    glColor3f(1, 1, 1);
    glBegin(GL_LINE_LOOP); glVertex2f(20, SH-40); glVertex2f(220, SH-40);
    glVertex2f(220, SH-20); glVertex2f(20, SH-20); glEnd();

    if (pHP < 30) {
        glEnable(GL_BLEND); glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
        glColor4f(1, 0, 0, 0.15f + 0.1f * sinf(glutGet(GLUT_ELAPSED_TIME) * 0.01f));
        glBegin(GL_QUADS); glVertex2f(0, 0); glVertex2f(SW, 0);
        glVertex2f(SW, SH); glVertex2f(0, SH); glEnd(); glDisable(GL_BLEND);
    }

    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();

    char buf[128];
    sprintf(buf, "HP: %d/%d", pHP, MAXHP); drawText2D(25, SH-60, buf, 1, 1, 1);
    sprintf(buf, "Score: %d", pScore); drawText2D(25, SH-85, buf, 1, 1, 0);
    sprintf(buf, "Level: %d", pLevel); drawText2D(25, SH-110, buf, 0.5f, 0.8f, 1);
    int aliveZ = 0;
    for (size_t i = 0; i < zombies.size(); i++) if (zombies[i].alive) aliveZ++;
    sprintf(buf, "Zombies: %d", aliveZ); drawText2D(25, SH-135, buf, 1, 0.3f, 0.3f);
    const char* wn[] = {"GUN", "KNIFE", "RIFLE"};
    sprintf(buf, "Weapon: %s Ammo: %d", wn[gWeap], pAmmo[gWeap]);
    drawText2D(25, SH-160, buf, 0.8f, 0.8f, 0.8f);
    sprintf(buf, "Lives: %d", pLives); drawText2D(25, SH-185, buf, 0.5f, 1, 0.5f);
    int mins = (int)pSurvTime / 60, secs = (int)pSurvTime % 60;
    sprintf(buf, "Time: %02d:%02d", mins, secs); drawText2D(SW-180, SH-30, buf, 0.8f, 0.8f, 0.8f);
    const char* cn[] = {"1st", "3rd", "Free"};
    sprintf(buf, "Cam: %s [C]", cn[gCam]); drawText2Dsmall(SW-220, SH-55, buf, 0.6f, 0.6f, 0.6f);
    if (!interactText.empty()) drawText2D(SW/2-100, 100, interactText.c_str(), 1, 1, 0);
    drawText2Dsmall(SW-280, 40, "WASD:Move Space:Jump Shift:Run", 0.5f, 0.5f, 0.5f);
    drawText2Dsmall(SW-280, 22, "E:Interact F:Flash 1/2/3:Weapon LClick:Shoot", 0.5f, 0.5f, 0.5f);
    drawWeaponFP();
}

// ======================= MENUS =======================
void drawMenu() {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glColor3f(0.05f, 0.02f, 0.08f);
    glBegin(GL_QUADS); glVertex2f(0, 0); glVertex2f(SW, 0); glVertex2f(SW, SH); glVertex2f(0, SH); glEnd();
    glColor3f(0.8f, 0.1f, 0.1f); glRasterPos2f(SW/2-180, SH-150);
    const char* title = "ZOMBIE SURVIVAL 3D";
    for (const char* c = title; *c; c++) glutBitmapCharacter(GLUT_BITMAP_TIMES_ROMAN_24, *c);
    glColor3f(0.5f, 0.5f, 0.5f); glRasterPos2f(SW/2-100, SH-190);
    const char* sub = "Survive the Apocalypse";
    for (const char* c = sub; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    const char* items[] = {"START GAME", "INSTRUCTIONS", "EXIT"};
    for (int i = 0; i < 3; i++) {
        if (i == menuSel) { glColor3f(1, 0.8f, 0);
            glBegin(GL_LINE_LOOP); glVertex2f(SW/2-120, SH/2-20-i*50); glVertex2f(SW/2+120, SH/2-20-i*50);
            glVertex2f(SW/2+120, SH/2+15-i*50); glVertex2f(SW/2-120, SH/2+15-i*50); glEnd();
        } else glColor3f(0.6f, 0.6f, 0.6f);
        glRasterPos2f(SW/2-60, SH/2-i*50);
        for (const char* c = items[i]; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, *c);
    }
    glColor3f(0.4f, 0.4f, 0.4f); glRasterPos2f(SW/2-150, 50);
    const char* ctrl = "Up/Down: Navigate  Enter: Select";
    for (const char* c = ctrl; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

void drawInstructions() {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glColor3f(0.05f, 0.02f, 0.08f);
    glBegin(GL_QUADS); glVertex2f(0, 0); glVertex2f(SW, 0); glVertex2f(SW, SH); glVertex2f(0, SH); glEnd();
    glColor3f(1, 0.8f, 0); glRasterPos2f(SW/2-80, SH-60);
    const char* t2 = "INSTRUCTIONS";
    for (const char* c = t2; *c; c++) glutBitmapCharacter(GLUT_BITMAP_TIMES_ROMAN_24, *c);
    const char* lines[] = {"WASD - Move  |  SPACE - Jump  |  SHIFT - Run",
        "Mouse - Look  |  Left Click - Shoot", "E - Interact (Doors/Switches)  |  F - Flashlight",
        "1/2/3 - Select Weapon (Gun/Knife/Rifle)", "C - Camera Mode  |  P - Pause", "",
        "Kill zombies (+10 pts). Enter buildings, toggle lights.", "Survive as long as possible!", "",
        "Press ESC or ENTER to go back"};
    for (int i = 0; i < 10; i++) {
        glColor3f(0.7f, 0.7f, 0.7f); glRasterPos2f(200, SH-120-i*30);
        for (const char* c = lines[i]; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    }
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

void drawPaused() {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glEnable(GL_BLEND); glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    glColor4f(0, 0, 0, 0.6f);
    glBegin(GL_QUADS); glVertex2f(0, 0); glVertex2f(SW, 0); glVertex2f(SW, SH); glVertex2f(0, SH);
    glEnd(); glDisable(GL_BLEND);
    glColor3f(1, 1, 0); glRasterPos2f(SW/2-50, SH/2+20);
    const char* p2 = "PAUSED";
    for (const char* c = p2; *c; c++) glutBitmapCharacter(GLUT_BITMAP_TIMES_ROMAN_24, *c);
    glColor3f(0.7f, 0.7f, 0.7f); glRasterPos2f(SW/2-100, SH/2-20);
    const char* r2 = "Press P to Resume, Q to Quit";
    for (const char* c = r2; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

void drawGameOver() {
    glMatrixMode(GL_PROJECTION); glPushMatrix(); glLoadIdentity(); glOrtho(0, SW, 0, SH, -1, 1);
    glMatrixMode(GL_MODELVIEW); glPushMatrix(); glLoadIdentity();
    glDisable(GL_LIGHTING); glDisable(GL_DEPTH_TEST);
    glColor3f(0.1f, 0, 0);
    glBegin(GL_QUADS); glVertex2f(0, 0); glVertex2f(SW, 0); glVertex2f(SW, SH); glVertex2f(0, SH); glEnd();
    glColor3f(1, 0, 0); glRasterPos2f(SW/2-100, SH/2+60);
    const char* go = "GAME OVER";
    for (const char* c = go; *c; c++) glutBitmapCharacter(GLUT_BITMAP_TIMES_ROMAN_24, *c);
    char buf[128];
    sprintf(buf, "Final Score: %d", pScore);
    glColor3f(1, 1, 0); glRasterPos2f(SW/2-70, SH/2+10);
    for (const char* c = buf; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, *c);
    sprintf(buf, "Level: %d", pLevel);
    glColor3f(0.7f, 0.7f, 0.7f); glRasterPos2f(SW/2-70, SH/2-20);
    for (const char* c = buf; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, *c);
    int mins = (int)pSurvTime / 60, secs = (int)pSurvTime % 60;
    sprintf(buf, "Survived: %02d:%02d", mins, secs);
    glRasterPos2f(SW/2-70, SH/2-50);
    for (const char* c = buf; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, *c);
    glColor3f(0.5f, 0.8f, 0.5f); glRasterPos2f(SW/2-100, SH/2-100);
    const char* rs = "ENTER: Restart  Q: Quit";
    for (const char* c = rs; *c; c++) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    glEnable(GL_DEPTH_TEST); glEnable(GL_LIGHTING);
    glMatrixMode(GL_PROJECTION); glPopMatrix(); glMatrixMode(GL_MODELVIEW); glPopMatrix();
}

// ======================= LIGHTING =======================
void setupLighting() {
    glEnable(GL_LIGHTING); glEnable(GL_LIGHT0); glEnable(GL_COLOR_MATERIAL);
    glColorMaterial(GL_FRONT_AND_BACK, GL_AMBIENT_AND_DIFFUSE);
    float sunAngle = dayTime * 2 * PI;
    float sunY = sinf(sunAngle), sunX = cosf(sunAngle);
    float intensity = fmax2(0.15f, sunY * 0.85f + 0.15f);
    GLfloat sunPos[4] = {sunX*80, fmax2(sunY*80, 5.0f), 30, 0};
    GLfloat sunAmb[4] = {intensity*0.4f, intensity*0.4f, intensity*0.45f, 1};
    GLfloat sunDif[4] = {intensity, intensity*0.95f, intensity*0.85f, 1};
    glLightfv(GL_LIGHT0, GL_POSITION, sunPos);
    glLightfv(GL_LIGHT0, GL_AMBIENT, sunAmb);
    glLightfv(GL_LIGHT0, GL_DIFFUSE, sunDif);

    if (flashlightOn) {
        glEnable(GL_LIGHT1);
        float fdx = sinf(pYaw)*cosf(pPitch), fdy = sinf(pPitch), fdz = -cosf(pYaw)*cosf(pPitch);
        GLfloat flPos[4] = {pPos.x, pPos.y+1.5f, pPos.z, 1};
        GLfloat flDif[4] = {0.9f, 0.9f, 0.8f, 1};
        GLfloat flDir[4] = {fdx, fdy, fdz, 0};
        glLightfv(GL_LIGHT1, GL_POSITION, flPos);
        glLightfv(GL_LIGHT1, GL_SPOT_DIRECTION, flDir);
        glLightf(GL_LIGHT1, GL_SPOT_CUTOFF, 30);
        glLightf(GL_LIGHT1, GL_SPOT_EXPONENT, 15);
        glLightfv(GL_LIGHT1, GL_DIFFUSE, flDif);
        glLightf(GL_LIGHT1, GL_CONSTANT_ATTENUATION, 0.5f);
        glLightf(GL_LIGHT1, GL_LINEAR_ATTENUATION, 0.05f);
    } else { glDisable(GL_LIGHT1); }

    for (size_t bi = 0; bi < buildings.size(); bi++) {
        Building& bld = buildings[bi];
        if (!bld.enterable) continue;
        for (size_t ri = 0; ri < bld.rooms.size(); ri++) {
            Room& r = bld.rooms[ri];
            if (r.lightOn) {
                glEnable(r.glLightID);
                GLfloat lp[4] = {r.lightPos.x, r.lightPos.y, r.lightPos.z, 1};
                GLfloat ld[4] = {0.9f, 0.85f, 0.7f, 1};
                GLfloat la[4] = {0.2f, 0.2f, 0.15f, 1};
                glLightfv(r.glLightID, GL_POSITION, lp);
                glLightfv(r.glLightID, GL_DIFFUSE, ld);
                glLightfv(r.glLightID, GL_AMBIENT, la);
                glLightf(r.glLightID, GL_CONSTANT_ATTENUATION, 0.8f);
                glLightf(r.glLightID, GL_LINEAR_ATTENUATION, 0.1f);
            } else { glDisable(r.glLightID); }
        }
    }
}

// ======================= GAME UPDATE =======================
void updateGame() {
    if (gState != PLAYING) return;

    int now = glutGet(GLUT_ELAPSED_TIME);
    float dt = (now - lastTime) / 1000.0f;
    if (dt > 0.1f) dt = 0.1f;
    if (dt <= 0) dt = 0.016f;
    lastTime = now;

    pSurvTime += dt;
    pShootCD -= dt; if (pShootCD < 0) pShootCD = 0;
    if (dayNightOn) dayTime += dt * 0.005f;
    if (dayTime > 1) dayTime -= 1;

    // ========== PLAYER MOVEMENT ==========
    bool running = isShiftHeld();
    float speed = (running ? PRUN : PWALK) * dt;

    Vec3 fwd(sinf(pYaw), 0, -cosf(pYaw));
    Vec3 right2(cosf(pYaw), 0, sinf(pYaw));
    Vec3 move(0, 0, 0);
    if (keys['w'] || keys['W']) move = move + fwd * speed;
    if (keys['s'] || keys['S']) move = move - fwd * speed;
    if (keys['a'] || keys['A']) move = move - right2 * speed;
    if (keys['d'] || keys['D']) move = move + right2 * speed;

    // Try full move
    Vec3 newPos = pPos + move;
    if (!checkCollision(newPos, 0.35f)) {
        pPos = newPos;
    } else {
        // Slide along X
        Vec3 newX = Vec3(newPos.x, pPos.y, pPos.z);
        if (!checkCollision(newX, 0.35f)) pPos = newX;
        else {
            // Slide along Z
            Vec3 newZ = Vec3(pPos.x, pPos.y, newPos.z);
            if (!checkCollision(newZ, 0.35f)) pPos = newZ;
        }
    }

    pPos.x = fmax2(-CHALF + 1.0f, fmin2(CHALF - 1.0f, pPos.x));
    pPos.z = fmax2(-CHALF + 1.0f, fmin2(CHALF - 1.0f, pPos.z));

    // Jump
    if (keys[' '] && pOnGround) { pVY = JUMP_V; pOnGround = false; }
    pVY += GRAVITY * dt;
    pPos.y += pVY * dt;
    if (pPos.y <= 0) { pPos.y = 0; pVY = 0; pOnGround = true; }

    // ========== BULLETS ==========
    for (size_t i = 0; i < bullets.size(); i++) {
        if (!bullets[i].active) continue;
        bullets[i].update(dt);
        for (size_t j = 0; j < zombies.size(); j++) {
            if (!zombies[j].alive) continue;
            if (bullets[i].pos.dist(zombies[j].pos) < 1.5f) {
                int dmg = (gWeap == W_RIFLE) ? RIFLEDMG : GUNDMG;
                zombies[j].hp -= dmg;
                if (zombies[j].hp <= 0) { zombies[j].alive = false; pScore += ZSCORE; }
                bullets[i].active = false; break;
            }
        }
        if (bullets[i].active && checkCollision(bullets[i].pos, 0.01f))
            bullets[i].active = false;
    }
    std::vector<Bullet> nb;
    for (size_t i = 0; i < bullets.size(); i++) {
        if (bullets[i].active) nb.push_back(bullets[i]);
    }
    bullets = nb;

    // ========== ZOMBIES ==========
    for (size_t i = 0; i < zombies.size(); i++) {
        zombies[i].update(pPos, dt);
        if (zombies[i].alive && zombies[i].attacking && zombies[i].atkTimer > 1.0f) {
            pHP -= ZDMG; zombies[i].atkTimer = 0;
            if (pHP <= 0) {
                pLives--;
                if (pLives <= 0) { gState = GAMEOVER; }
                else { pHP = MAXHP; pPos = Vec3(0, 0, 5); }
            }
        }
    }
    std::vector<Zombie> nz;
    for (size_t i = 0; i < zombies.size(); i++) {
        if (zombies[i].alive || zombies[i].dieTimer < 2.0f) nz.push_back(zombies[i]);
    }
    zombies = nz;

    int alive = 0;
    for (size_t i = 0; i < zombies.size(); i++) if (zombies[i].alive) alive++;
    if (alive == 0) {
        pLevel++; pScore += pLevel * 20;
        for (int i = 0; i < 3 + pLevel * 2; i++) spawnZombie();
    }

    // ========== DOORS ==========
    for (size_t bi = 0; bi < buildings.size(); bi++) {
        for (size_t di = 0; di < buildings[bi].doors.size(); di++)
            buildings[bi].doors[di].update(dt);
    }

    // ========== REBUILD COLLIDERS ==========
    wallColliders.clear();
    for (size_t i = 0; i < buildings.size(); i++) {
        std::vector<AABB> c3 = buildings[i].allColliders();
        wallColliders.insert(wallColliders.end(), c3.begin(), c3.end());
    }
    for (size_t i = 0; i < carPositions.size(); i++) {
        float cx = carPositions[i].x, cz = carPositions[i].z;
        wallColliders.push_back(AABB(Vec3(cx-1.2f, 0, cz-2.5f), Vec3(cx+1.2f, 1.5f, cz+2.5f)));
    }
    wallColliders.push_back(AABB(Vec3(-CHALF, 0,-CHALF), Vec3(CHALF, 4,-CHALF+0.5f)));
    wallColliders.push_back(AABB(Vec3(-CHALF, 0, CHALF-0.5f), Vec3(CHALF, 4, CHALF)));
    wallColliders.push_back(AABB(Vec3(-CHALF, 0,-CHALF), Vec3(-CHALF+0.5f, 4, CHALF)));
    wallColliders.push_back(AABB(Vec3(CHALF-0.5f, 0,-CHALF), Vec3(CHALF, 4, CHALF)));

    checkInteractions();

    bonusTimer += dt;
    if (bonusTimer > 10) { pScore += 5; bonusTimer = 0; }
}

// ======================= DISPLAY =======================
void display() {
    glClearColor(0, 0, 0, 1);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    // *** CRITICAL: UPDATE GAME STATE EVERY FRAME ***
    updateGame();

    if (gState == MENU) {
        if (showInstructions) drawInstructions(); else drawMenu();
        glutSwapBuffers(); return;
    }
    if (gState == GAMEOVER) { drawGameOver(); glutSwapBuffers(); return; }

    glMatrixMode(GL_PROJECTION); glLoadIdentity();
    gluPerspective(60, (double)SW / (double)SH, 0.1, 200);
    glMatrixMode(GL_MODELVIEW); glLoadIdentity();

    Vec3 camPos, lookAt;
    float ddx = sinf(pYaw) * cosf(pPitch);
    float ddy = sinf(pPitch);
    float ddz = -cosf(pYaw) * cosf(pPitch);
    lookAt = pPos + Vec3(0, 1.6f, 0) + Vec3(ddx, ddy, ddz);

    if (gCam == FIRST) {
        camPos = pPos + Vec3(0, 1.6f, 0);
    } else if (gCam == THIRD) {
        camPos = pPos + Vec3(0, 2.5f, 0) - Vec3(ddx, 0, ddz) * camDist;
        if (camPos.y < 0.5f) camPos.y = 0.5f;
    } else {
        float fx = sinf(camYaw) * cosf(camPitch * PI / 180.0f);
        float fy = sinf(camPitch * PI / 180.0f);
        float fz = -cosf(camYaw) * cosf(camPitch * PI / 180.0f);
        camPos = pPos + Vec3(0, 3, 0);
        lookAt = camPos + Vec3(fx, fy, fz);
    }
    gluLookAt(camPos.x, camPos.y, camPos.z, lookAt.x, lookAt.y, lookAt.z, 0, 1, 0);

    glEnable(GL_FOG);
    float fogCol = fmax2(0.15f, sinf(dayTime * 2 * PI) * 0.3f + 0.2f);
    GLfloat fc[4] = {fogCol*0.5f, fogCol*0.5f, fogCol*0.6f, 1};
    glFogfv(GL_FOG_COLOR, fc); glFogi(GL_FOG_MODE, GL_EXP2); glFogf(GL_FOG_DENSITY, 0.012f);

    setupLighting(); glEnable(GL_DEPTH_TEST); glEnable(GL_NORMALIZE);

    drawSkybox(); drawGround();
    for (size_t i = 0; i < buildings.size(); i++) drawBuilding(buildings[i]);
    for (size_t i = 0; i < treePositions.size(); i++) drawTree(treePositions[i]);
    for (size_t i = 0; i < streetLights.size(); i++) drawStreetLight(streetLights[i]);
    for (size_t i = 0; i < carPositions.size(); i++) drawCar(carPositions[i], carRots[i], carColors[i]);

    // Player in 3rd person
    if (gCam != FIRST) {
        glPushMatrix(); glTranslatef(pPos.x, pPos.y, pPos.z); glRotatef(pYaw * 180.0f / PI, 0, 1, 0);
        drawHumanoid(0.2f, 0.3f, 0.8f, 0, false, 1.0f);
        setColor(0.3f, 0.3f, 0.3f);
        glPushMatrix(); glTranslatef(0.35f, 1.1f,-0.3f); drawBox(0.08f, 0.08f, 0.4f); glPopMatrix();
        glPopMatrix();
    }

    for (size_t i = 0; i < zombies.size(); i++) drawZombie(zombies[i]);

    // Bullets - BIGGER so you can see them
    for (size_t i = 0; i < bullets.size(); i++) {
        if (!bullets[i].active) continue;
        // Yellow glowing bullet
        setColor(1, 1, 0);
        glPushMatrix(); glTranslatef(bullets[i].pos.x, bullets[i].pos.y, bullets[i].pos.z);
        glutSolidSphere(0.15f, 8, 8); glPopMatrix();
        // Trail
        setColor(1, 0.5f, 0);
        Vec3 tail = bullets[i].pos - bullets[i].dir * 0.8f;
        glPushMatrix(); glTranslatef(tail.x, tail.y, tail.z);
        glutSolidSphere(0.08f, 6, 6); glPopMatrix();
    }

    glDisable(GL_FOG);
    drawHUD();
    if (gState == PAUSED) drawPaused();

    glutSwapBuffers();
}

// ======================= INPUT =======================
void keyboard(unsigned char key, int x, int y) {
    keys[key] = true;

    if (gState == MENU) {
        if (showInstructions) {
            if (key == 27 || key == 13) showInstructions = false;
            return;
        }
        if (key == 13 && menuSel == 0) {
            gState = PLAYING; lastTime = glutGet(GLUT_ELAPSED_TIME);
            pHP = MAXHP; pScore = 0; pLives = 3; pLevel = 1; pSurvTime = 0;
            pPos = Vec3(0, 0, 5); pYaw = 0; pPitch = 0; initWorld();
        }
        if (key == 13 && menuSel == 1) showInstructions = true;
        if (key == 13 && menuSel == 2) exit(0);
        return;
    }
    if (gState == GAMEOVER) {
        if (key == 13) { gState = MENU; menuSel = 0; }
        if (key == 'q' || key == 'Q') exit(0);
        return;
    }
    if (gState == PAUSED) {
        if (key == 'p' || key == 'P') { gState = PLAYING; lastTime = glutGet(GLUT_ELAPSED_TIME); }
        if (key == 'q' || key == 'Q') exit(0);
        return;
    }

    switch (key) {
        case 'p': case 'P': gState = PAUSED; break;
        case 'c': case 'C': gCam = (CamMode)((gCam + 1) % 3); break;
        case 'f': case 'F': flashlightOn = !flashlightOn; break;
        case 'e': case 'E':
            if (nearDoor) nearDoor->toggle();
            if (nearSwitch) nearSwitch->lightOn = !nearSwitch->lightOn;
            break;
        case '1': gWeap = W_GUN; break;
        case '2': gWeap = W_KNIFE; break;
        case '3': gWeap = W_RIFLE; break;
        case 27: exit(0); break;
    }
}

void keyboardUp(unsigned char key, int x, int y) {
    keys[key] = false;
}

void specialKeys(int key, int x, int y) {
    if (gState == MENU && !showInstructions) {
        if (key == GLUT_KEY_UP) menuSel = (menuSel + 2) % 3;
        if (key == GLUT_KEY_DOWN) menuSel = (menuSel + 1) % 3;
    }
}

void specialKeysUp(int key, int x, int y) {
    // Nothing needed
}

void mouseMotion(int x, int y) {
    if (gState != PLAYING) return;
    int cx = SW / 2, cy = SH / 2;
    int mdx = x - cx, mdy = y - cy;
    pYaw   += mdx * MOUSE_SENS;
    pPitch -= mdy * MOUSE_SENS;
    if (pPitch > 1.2f) pPitch = 1.2f;
    if (pPitch < -1.2f) pPitch = -1.2f;
    glutWarpPointer(cx, cy);
}

void mouseClick(int button, int state, int x, int y) {
    if (gState != PLAYING) return;
    if (button == GLUT_LEFT_BUTTON && state == GLUT_DOWN) shoot();
}

void passiveMotion(int x, int y) {
    if (gState != PLAYING) return;
    int cx = SW / 2, cy = SH / 2;
    int mdx = x - cx, mdy = y - cy;
    pYaw   += mdx * MOUSE_SENS;
    pPitch -= mdy * MOUSE_SENS;
    if (pPitch > 1.2f) pPitch = 1.2f;
    if (pPitch < -1.2f) pPitch = -1.2f;
    glutWarpPointer(cx, cy);
}

// ======================= MAIN =======================
int main(int argc, char** argv) {
    glutInit(&argc, argv);
    glutInitWindowSize(SW, SH);
    glutInitDisplayMode(GLUT_RGB | GLUT_DOUBLE | GLUT_DEPTH);
    glutCreateWindow("Zombie Survival 3D - OpenGL");

    glutDisplayFunc(display);
    glutIdleFunc(display);
    glutKeyboardFunc(keyboard);
    glutKeyboardUpFunc(keyboardUp);
    glutSpecialFunc(specialKeys);
    glutMouseFunc(mouseClick);
    glutMotionFunc(mouseMotion);
    glutPassiveMotionFunc(passiveMotion);

    glutSetCursor(GLUT_CURSOR_NONE);
    glutWarpPointer(SW / 2, SH / 2);

    glEnable(GL_DEPTH_TEST); glDepthFunc(GL_LESS);
    glEnable(GL_NORMALIZE); glShadeModel(GL_SMOOTH);
    glClearColor(0, 0, 0, 1);

    initWorld();

    glutMainLoop();
    return 0;
}

