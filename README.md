# app.py
from dataclasses import dataclass
from typing import List, Optional, Dict, Set
from abc import ABC, abstractmethod
import os
import json

from flask import (
    Flask, render_template, request,
    redirect, url_for, session, flash
)

app = Flask(__name__)
app.secret_key = "32234039"

@dataclass
class Pokemon:
    id: int
    name: str
    types: List[str]
    hp : int
    attack : int
    defense : int
    image_path: Optional[str] = None #이미지 경로

    def type_str(self) -> str:
        return ", ".join(self.types)  #타입 문자열 반환


class MasterPokedex:
    def __init__(self) -> None:
        self._pokemons: Dict[int, Pokemon] = {}
        self._build()

    def _build(self) -> None:
        def img(num: int, eng: str) -> str:
            return f"/static/images/{num:03}_{eng}.png"

        data = [
            Pokemon(1,  "이상해씨", ["grass", "poison"],     45, 49, 49, img(1,  "bulbasaur")),
            Pokemon(2,  "이상해풀", ["grass", "poison"],     60, 62, 63, img(2,  "ivysaur")),
            Pokemon(3,  "이상해꽃", ["grass", "poison"],     80, 82, 83, img(3,  "venusaur")),
            Pokemon(4,  "파이리",   ["fire"],         39, 52, 43, img(4,  "charmander")),
            Pokemon(5,  "리자드",   ["fire"],         58, 64, 58, img(5,  "charmeleon")),
            Pokemon(6,  "리자몽",   ["fire", "flying"], 78, 84, 78, img(6,  "charizard")),
            Pokemon(7,  "꼬부기",   ["water"],           44, 48, 65, img(7,  "squirtle")),
            Pokemon(8,  "어니부기", ["water"],           59, 63, 80, img(8,  "wartortle")),
            Pokemon(9,  "거북왕",   ["water"],           79, 83, 100, img(9, "blastoise")),
            Pokemon(172, "피츄",   ["electric"],   45, 30, 35, img(172, "pichu")),
            Pokemon(25, "피카츄", ["electric"],   35, 55, 40, img(25, "pikachu")),
            Pokemon(26, "라이츄", ["electric"],   60, 90, 55, img(26, "raichu")),
            Pokemon(702, "데덴네", ["electric", "fairy"], 67, 58, 57, img(702, "dedenne")),
            Pokemon(495, "주리비얀", ["grass"],    60, 60, 75, img(495, "snivy")),
            Pokemon(496, "샤비",   ["grass"],      75, 75, 95, img(496, "servine")),
            Pokemon(497, "샤로다", ["grass"],      75, 75, 95, img(497, "serperior")),
            Pokemon(501, "수댕이", ["water"],      55, 55, 45, img(501, "oshawott")),
            Pokemon(502, "쌍검자비", ["water"],    75, 75, 60, img(502, "dewott")),
            Pokemon(503, "대검귀", ["water"],      95, 100, 85, img(503, "samurott")),
            Pokemon(448, "루카리오", ["fighting", "steel"], 70, 110, 70, img(448, "lucario")),
        ]
        self._pokemons = {p.id: p for p in data}

    def all(self) -> List[Pokemon]:
        return list(self._pokemons.values())

    def get(self, pid: int) -> Optional[Pokemon]:
        return self._pokemons.get(pid)

    def exists(self, pid: int) -> bool:
        return pid in self._pokemons

    def size(self) -> int:
        return len(self._pokemons)


class UserDex:
    def __init__(self, username: str, owned_ids: Optional[Set[int]] = None) -> None:
        self.username = username
        self.owned_ids: Set[int] = owned_ids if owned_ids is not None else set()

    def catch(self, pid: int, master: MasterPokedex) -> str:
        if not master.exists(pid):
            return "그 번호의 포켓몬은 존재하지 않습니다!"
        if pid in self.owned_ids:
            p = master.get(pid)
            return f"{p.name} 은(는) 이미 도감에 보유 중입니다!"
        self.owned_ids.add(pid)
        p = master.get(pid)
        return f"신난다~ {p.name} 을(를) 잡았다!"
    
    def release(self, pid: int, master: MasterPokedex) -> str:
        if pid not in self.owned_ids:
            return "아직 포획하지 않은 포켓몬입니다."
        self.owned_ids.remove(pid)
        p = master.get(pid)
        return f"{p.name} 을(를) 도감에서 삭제했습니다."

    def progress(self, master: MasterPokedex) -> float:
        if master.size() == 0:
            return 0.0
        return len(self.owned_ids) / master.size()

    def to_dict(self) -> dict:
        return {
            "username": self.username,
            "owned_ids": sorted(list(self.owned_ids)),
        }

    @classmethod
    def from_dict(cls, data: dict) -> "UserDex":
        username = data.get("username", "unknown")
        owned_ids = set(data.get("owned_ids", []))
        return cls(username, owned_ids)


# -----------------------------
# 저장소 (Strategy)
# -----------------------------
class DexStorage(ABC):
    @abstractmethod
    def load(self, username: str) -> UserDex:
        ...

    @abstractmethod
    def save(self, userdex: UserDex) -> None:
        ...


class JsonDexStorage(DexStorage):
    def __init__(self, base_dir: str = "data") -> None:
        self.base_dir = base_dir
        os.makedirs(self.base_dir, exist_ok=True)

    def _filename(self, username: str) -> str:
        safe = username.replace(" ", "_")
        return os.path.join(self.base_dir, f"{safe}_dex.json")

    def load(self, username: str) -> UserDex:
        path = self._filename(username)
        if not os.path.exists(path):
            return UserDex(username)

        try:
            with open(path, "r", encoding="utf-8") as f:
                data = json.load(f)
            return UserDex.from_dict(data)
        except Exception:
            return UserDex(username)

    def save(self, userdex: UserDex) -> None:
        path = self._filename(userdex.username)
        with open(path, "w", encoding="utf-8") as f:
            json.dump(userdex.to_dict(), f, ensure_ascii=False, indent=2)


# 전역 싱글턴처럼 사용
master = MasterPokedex()
storage = JsonDexStorage(base_dir="data")


# -----------------------------
# 헬퍼 함수
# -----------------------------
def get_current_username() -> Optional[str]:
    return session.get("username")


def get_current_userdex() -> Optional[UserDex]:
    username = get_current_username()
    if not username:
        return None
    return storage.load(username)


# -----------------------------
# 라우트
# -----------------------------
@app.route("/", methods=["GET"])
def index():
    # 로그인 페이지
    return render_template("index.html")


@app.route("/login", methods=["POST"])
def login():
    username = request.form.get("username", "").strip()
    if not username:
        flash("유저 이름을 입력하세요.", "error")
        return redirect(url_for("index"))

    session["username"] = username
    # 유저 도감이 없으면 새로 생성됨
    _ = storage.load(username)
    flash(f"포켓몬 트레이너 {username}님, 환영합니다!", "success")
    return redirect(url_for("pokedex"))


@app.route("/logout")
def logout():
    session.pop("username", None)
    flash("로그아웃되었습니다.", "info")
    return redirect(url_for("index"))


@app.route("/pokedex")
def pokedex():
    userdex = get_current_userdex()
    if userdex is None:
        flash("먼저 로그인하세요.", "error")
        return redirect(url_for("index"))

    q = request.args.get("q", "").strip()
    view = request.args.get("view", "all")

    pokemons = sorted(master.all(), key=lambda p: p.id)

    if q:
        lower_q = q.lower()
        filtered = []
        for p in pokemons:
            # 이름, 번호, 타입 검색
            if lower_q in p.name.lower():
                filtered.append(p)
                continue
            if lower_q in str(p.id):
                filtered.append(p)
                continue
            
            if any(lower_q in t.lower() for t in p.types):
                filtered.append(p)
        pokemons = filtered

    owned_ids = userdex.owned_ids

    if view == "owned":
        pokemons = [p for p in pokemons if p.id in owned_ids]
    elif view == "unowned":
        pokemons = [p for p in pokemons if p.id not in owned_ids]

    progress = userdex.progress(master)
    return render_template(
        "pokedex.html",
        username=userdex.username,
        pokemons=pokemons,
        owned_ids=owned_ids,
        progress=progress,
        have_count=len(owned_ids),
        total_count=master.size(),
        q=q,
        view=view,
    )

@app.route("/pokemon/<int:pid>")
def pokemon_detail(pid: int):
    userdex = get_current_userdex()
    if userdex is None:
        flash("먼저 로그인하세요.", "error")
        return redirect(url_for("index"))

    pokemon = master.get(pid)
    if pokemon is None:
        flash("해당 번호의 포켓몬이 존재하지 않습니다.", "error")
        return redirect(url_for("pokedex"))

    owned = pid in userdex.owned_ids
    return render_template(
        "pokemon_detail.html",
        username=userdex.username,
        pokemon=pokemon,
        owned=owned,
    )

@app.route("/catch/<int:pid>", methods=["POST"])
def catch(pid: int):
    userdex = get_current_userdex()
    if userdex is None:
        flash("먼저 로그인하세요.", "error")
        return redirect(url_for("index"))

    msg = userdex.catch(pid, master)
    storage.save(userdex)
    flash(msg, "info")
    return redirect(url_for("pokedex"))

@app.route("/release/<int:pid>", methods=["POST"])
def release(pid: int):
    userdex = get_current_userdex()
    if userdex is None:
        flash("먼저 로그인하세요.", "error")
        return redirect(url_for("index"))
    
    msg = userdex.release(pid, master)
    storage.save(userdex)
    flash(msg, "warning")
    return redirect(url_for("pokedex"))

if __name__ == "__main__":
    app.run(debug=True)
