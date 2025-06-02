#!/usr/bin/env python3
"""
CARLA 센서 제거·재생성 툴
  • 센서를 선택해 destroy
  • 같은 차량(ID 기본 hero)에 원래 위치·회전으로 다시 spawn
"""

import carla
import time
import math

# ─────────────────────────  좌표계 보조 함수  ──────────────────────────
def to_rad(rot):
    return math.radians(rot.roll), math.radians(rot.pitch), math.radians(rot.yaw)

def world_vec_to_vehicle(vec, veh_rot):
    """월드 기준 벡터 → 차량 local 기준 벡터"""
    r, p, y = to_rad(veh_rot)
    cy, sy, cp, sp, cr, sr = math.cos(y), math.sin(y), math.cos(p), math.sin(p), math.cos(r), math.sin(r)

    # ZYX 회전 행렬의 전치(Rᵀ)·곱
    return carla.Location(
        x =  cy*cp*vec.x + sy*cp*vec.y - sp*vec.z,
        y = (cy*sp*sr - sy*cr)*vec.x + (sy*sp*sr + cy*cr)*vec.y + cp*sr*vec.z,
        z = (cy*sp*cr + sy*sr)*vec.x + (sy*sp*cr - cy*sr)*vec.y + cp*cr*vec.z)

def get_relative_transform(sensor_t, vehicle_t):
    """월드 Transform → 차량 local Transform"""
    delta_loc = sensor_t.location - vehicle_t.location
    rel_loc   = world_vec_to_vehicle(delta_loc, vehicle_t.rotation)

    rel_rot = carla.Rotation(
        pitch = sensor_t.rotation.pitch - vehicle_t.rotation.pitch,
        yaw   = sensor_t.rotation.yaw   - vehicle_t.rotation.yaw,
        roll  = sensor_t.rotation.roll  - vehicle_t.rotation.roll)

    return carla.Transform(rel_loc, rel_rot)

# ──────────────────────────────  메인  ───────────────────────────────
def main():
    client = carla.Client('localhost', 2000)
    client.set_timeout(10.0)
    world = client.get_world()
    blueprints = world.get_blueprint_library()

    removed = {}   # {sensor_id: {type_id, rel_tf, attrs}}
    time.sleep(1)  # world 초기화 딜레이

    while True:
        print("\n=== 현재 액터 목록 ===")
        sensors  = world.get_actors().filter('sensor.*')
        vehicles = world.get_actors().filter('vehicle.*')

        for s in sensors:
            mark = '[X]' if s.id in removed else ''
            print(f"{s.id:<4} {s.type_id:<30} {s.attributes.get('role_name', '')} {mark}")

        hero_id = None
        for v in vehicles:
            if 'hero' in v.attributes.get('role_name', ''):
                hero_id = v.id
            print(f"{v.id:<4} {v.type_id:<30} {v.attributes.get('role_name', '')}")

        usr = input(f"\n➜ 센서 ID 제거,  r <ID> 재생성 (대상차량 {hero_id}),  q 종료: ").strip()
        if usr.lower() == 'q':
            break

        try:
            # ───── 재생성 ─────
            if usr.startswith('r '):
                sid = int(usr.split()[1])
                if sid not in removed:
                    print("제거 목록에 없음.")
                    continue

                info = removed[sid]
                bp   = blueprints.find(info['type_id'])
                for k, v in info['attrs'].items():
                    if bp.has_attribute(k):
                        bp.set_attribute(k, v)

                vehicle = world.get_actor(hero_id)
                if vehicle is None:
                    print("대상 차량을 찾을 수 없음.")
                    continue

                new_sensor = world.spawn_actor(bp, info['rel_tf'], attach_to=vehicle)
                print(f"✅ 리스폰 완료 (새 ID {new_sensor.id})")
                continue

            # ───── 제거 ─────
            sid   = int(usr)
            actor = world.get_actor(sid)
            if actor is None or 'sensor.' not in actor.type_id:
                print("해당 ID는 센서가 아님.")
                continue

            parent = actor.parent
            if parent is None:
                print("센서가 차량에 붙어있지 않음.")
                continue

            removed[sid] = {
                'type_id': actor.type_id,
                'rel_tf' : get_relative_transform(actor.get_transform(), parent.get_transform()),
                'attrs'  : dict(actor.attributes)
            }
            actor.destroy()
            print("✅ 제거 완료.")

        except Exception as e:
            print("오류:", e)

if __name__ == '__main__':
    main()

