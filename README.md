## 사용 예시

```python
# 포켓몬 기본 정보 출력
pokemons = [
    {"id": 1,  "name": "이상해씨", "types": ["grass", "poison"], "hp": 45},
    {"id": 4,  "name": "파이리",   "types": ["fire"],            "hp": 39},
    {"id": 7,  "name": "꼬부기",   "types": ["water"],           "hp": 44},
    {"id": 25, "name": "피카츄",   "types": ["electric"],        "hp": 35},
    {"id": 448,"name": "루카리오", "types": ["fighting","steel"],"hp": 70},
]
for p in pokemons:
    types = ", ".join(p["types"])
    print(f"No.{p['id']:03d} {p['name']} | 타입: {types} | HP: {p['hp']}")

# 도감 포획 진행률 계산
total = 20
owned = [1, 4, 7, 25]
progress = len(owned) / total * 100
print(f"보유: {len(owned)}마리 / 전체: {total}마리")
print(f"도감 완성률: {progress:.1f}%")
```
