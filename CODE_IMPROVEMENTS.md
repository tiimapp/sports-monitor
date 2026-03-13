# 💻 Sports Monitor - 代码改进建议 v3.0.7

## 问题分析

### 当前问题

1. **输出格式不统一**
   - 篮球、足球、羽毛球使用不同的格式化函数
   - 纯文本输出，非表格化
   - 信息密度低，不易阅读

2. **缺少直播平台信息**
   - `format_basketball_game()` 和 `format_football_fixture()` 未包含直播平台
   - 数据源解析时未提取直播信息

3. **足球/羽毛球信息不完整**
   - 足球缺少比赛状态（未开始/进行中/已结束）
   - 羽毛球缺少轮次信息（预赛/1/8 决赛/半决赛/决赛）

4. **代码重复**
   - 每个运动种类都有类似的输出逻辑
   - 缺少统一的表格渲染函数

---

## 改进建议

### 1. 新增表格渲染函数

**位置**: `sports_monitor.py` 第 850-900 行之间

```python
# ============================================================================
# 表格渲染（v3.0.7 新增）
# ============================================================================

def render_ascii_table(headers: List[str], rows: List[List[str]], 
                       col_alignments: List[str] = None) -> str:
    """
    渲染 ASCII 表格（v3.0.7 新增）
    
    Args:
        headers: 表头列表
        rows: 数据行列表
        col_alignments: 列对齐方式 ['left', 'center', 'right']，默认 left
    
    Returns:
        格式化的表格字符串
    """
    if not rows:
        return "📌 暂无数据"
    
    # 计算每列最大宽度
    col_widths = [len(h) for h in headers]
    for row in rows:
        for i, cell in enumerate(row):
            if i < len(col_widths):
                col_widths[i] = max(col_widths[i], len(str(cell)))
    
    # 默认左对齐
    if col_alignments is None:
        col_alignments = ['left'] * len(headers)
    
    def align_text(text: str, width: int, alignment: str) -> str:
        if alignment == 'center':
            return text.center(width)
        elif alignment == 'right':
            return text.rjust(width)
        else:  # left
            return text.ljust(width)
    
    # 构建表格
    lines = []
    
    # 顶线
    lines.append('┌' + '┬'.join('─' * (w + 2) for w in col_widths) + '┐')
    
    # 表头
    header_cells = [align_text(h, w, 'center') for h, w in zip(headers, col_widths)]
    lines.append('│' + '│'.join(f' {c} ' for c in header_cells) + '│')
    
    # 分隔线
    lines.append('├' + '┼'.join('─' * (w + 2) for w in col_widths) + '┤')
    
    # 数据行
    for row in rows:
        # 填充缺失列
        while len(row) < len(headers):
            row.append('')
        row_cells = [align_text(str(c), w, a) for c, w, a in zip(row, col_widths, col_alignments)]
        lines.append('│' + '│'.join(f' {c} ' for c in row_cells) + '│')
    
    # 底线
    lines.append('└' + '┴'.join('─' * (w + 2) for w in col_widths) + '┘')
    
    return '\n'.join(lines)


def render_simple_table(headers: List[str], rows: List[List[str]]) -> str:
    """
    简化表格（用于 Discord/微信等平台）
    
    Args:
        headers: 表头列表
        rows: 数据行列表
    
    Returns:
        简化格式的表格字符串
    """
    lines = []
    for row in rows:
        lines.append(f"【{headers[0]}】{row[0] if len(row) > 0 else ''}")
        if len(row) > 1:
            lines.append(f"【{headers[1]}】{row[1]}")
        if len(row) > 2:
            lines.append(f"【{headers[2]}】{row[2]}")
        if len(row) > 3:
            lines.append(f"【{headers[3]}】{row[3]}")
        lines.append("─" * 40)
    return '\n'.join(lines)
```

---

### 2. 重构篮球比赛格式化函数

**位置**: `sports_monitor.py` 第 888 行附近

**当前代码**:
```python
def format_basketball_game(game: dict, favorite_teams: List[str], priority: str = 'medium') -> str:
    """格式化篮球比赛"""
    home = game.get('home', 'Unknown')
    away = game.get('away', 'Unknown')
    time_beijing = game.get('time', '')
    arena = game.get('arena', '')
    
    # ... 省略评分逻辑 ...
    
    return f"{stars} {time_str} | {away} @ {home}\n   📍 {arena}\n   🔥 推荐指数：{score}/100"
```

**改进后**:
```python
def format_basketball_game(game: dict, favorite_teams: List[str], 
                          priority: str = 'medium') -> dict:
    """
    格式化篮球比赛（v3.0.7 返回结构化数据）
    
    Args:
        game: 比赛数据字典
        favorite_teams: 关注球队列表
        priority: 优先级
    
    Returns:
        结构化比赛数据字典
    """
    home = game.get('home', 'Unknown')
    away = game.get('away', 'Unknown')
    time_beijing = game.get('time', '')
    arena = game.get('arena', '')
    broadcast = game.get('broadcast', ['待定'])
    
    # 解析时间
    try:
        dt = date_parser.parse(time_beijing)
        time_str = dt.strftime('%m-%d %H:%M')
    except:
        time_str = time_beijing[:16].replace('T', ' ') if time_beijing else 'TBD'
    
    # 计算评分
    score = 50
    if priority == 'high':
        score += 10
    if home in favorite_teams or away in favorite_teams:
        score += 30
    
    # 确定优先级标记
    priority_mark = '💚' if (home in favorite_teams or away in favorite_teams) else ''
    
    return {
        'time': time_str,
        'matchup': f"{priority_mark} {away} @ {home}".strip(),
        'competition': 'NBA 常规赛',
        'broadcast': '、'.join(broadcast) if isinstance(broadcast, list) else broadcast,
        'arena': arena,
        'score': score,
        'stars': format_star_rating(score)
    }
```

---

### 3. 重构足球比赛格式化函数

**位置**: `sports_monitor.py` 第 912 行附近

**改进后**:
```python
def format_football_fixture(fixture: dict, favorite_teams: List[str]) -> dict:
    """
    格式化足球比赛（v3.0.7 返回结构化数据）
    
    Args:
        fixture: 比赛数据字典
        favorite_teams: 关注球队列表
    
    Returns:
        结构化比赛数据字典
    """
    time = fixture.get('time', 'TBD')
    home = fixture.get('home', 'Unknown')
    away = fixture.get('away', 'Unknown')
    score = fixture.get('score', '')
    status = fixture.get('status', '')
    note = fixture.get('note', '')
    priority = fixture.get('priority', 'medium')
    broadcast = fixture.get('broadcast', ['待定'])
    round_info = fixture.get('round', '')  # 轮次信息
    
    # 计算评分
    score_val = 50
    if priority == 'high':
        score_val += 20
    if any(t in home for t in favorite_teams) or any(t in away for t in favorite_teams):
        score_val += 25
    
    # 确定优先级标记
    priority_mark = '💚' if (any(t in home for t in favorite_teams) or 
                           any(t in away for t in favorite_teams)) else ''
    
    # 构建比赛状态
    if score and status:
        time_str = f"{time} ({status})"
        match_status = status
    else:
        time_str = time
        match_status = '未开始'
    
    return {
        'time': time_str,
        'matchup': f"{priority_mark} {home} vs {away}".strip(),
        'competition': round_info or '足球比赛',
        'broadcast': '、'.join(broadcast) if isinstance(broadcast, list) else broadcast,
        'status': match_status,
        'score': score_val,
        'stars': format_star_rating(score_val)
    }
```

---

### 4. 新增羽毛球比赛格式化函数

**位置**: `sports_monitor.py` 第 980 行附近（新增）

```python
def format_badminton_match(match: dict, favorite_players: List[str]) -> dict:
    """
    格式化羽毛球比赛（v3.0.7 新增）
    
    Args:
        match: 比赛数据字典
        favorite_players: 关注选手列表
    
    Returns:
        结构化比赛数据字典
    """
    time = match.get('time', 'TBD')
    description = match.get('description', '')
    tournament = match.get('tournament', '')
    round_info = match.get('round', '')  # 轮次
    broadcast = match.get('broadcast', ['待定'])
    event_type = match.get('event_type', '')  # 男单/女双等
    
    # 解析选手
    players = []
    if 'vs' in description:
        players = description.split('vs')
    elif '对阵' in description:
        players = description.split('对阵')
    
    # 检查是否有关注选手
    is_favorite = any(p in description for p in favorite_players)
    priority_mark = '💚' if is_favorite else ''
    
    # 计算评分
    score_val = 60
    if is_favorite:
        score_val += 25
    if '决赛' in round_info:
        score_val += 15
    elif '半决赛' in round_info:
        score_val += 10
    
    return {
        'time': time,
        'matchup': f"{priority_mark} {description[:50]}".strip(),
        'competition': tournament,
        'broadcast': '、'.join(broadcast) if isinstance(broadcast, list) else broadcast,
        'round': round_info or '正赛',
        'event_type': event_type,
        'score': score_val,
        'stars': format_star_rating(score_val)
    }
```

---

### 5. 新增电竞比赛格式化函数

**位置**: `sports_monitor.py` 第 1020 行附近（新增）

```python
def format_esports_match(match: dict, favorite_teams: List[str]) -> dict:
    """
    格式化电竞比赛（v3.0.7 新增）
    
    Args:
        match: 比赛数据字典
        favorite_teams: 关注战队列表
    
    Returns:
        结构化比赛数据字典
    """
    time = match.get('time', 'TBD')
    description = match.get('description', '')
    league = match.get('league', '')
    bo_series = match.get('bo', 'BO3')  # BO3/BO5
    broadcast = match.get('broadcast', ['待定'])
    
    # 检查是否有关注战队
    is_favorite = any(t in description for t in favorite_teams)
    priority_mark = '💚' if is_favorite else ''
    
    # 计算评分
    score_val = 60
    if is_favorite:
        score_val += 25
    if '决赛' in description:
        score_val += 15
    
    return {
        'time': time,
        'matchup': f"{priority_mark} {description[:50]}".strip(),
        'competition': league,
        'broadcast': '、'.join(broadcast) if isinstance(broadcast, list) else broadcast,
        'format': bo_series,
        'score': score_val,
        'stars': format_star_rating(score_val)
    }
```

---

### 6. 新增直播平台映射函数

**位置**: `sports_monitor.py` 第 850 行附近（新增）

```python
# ============================================================================
# 直播平台映射（v3.0.7 新增）
# ============================================================================

BROADCAST_PLATFORMS = {
    'nba': ['CCTV5', '腾讯体育'],
    'cba': ['CCTV5+', '咪咕体育'],
    'premier_league': ['爱奇艺体育', '咪咕体育'],
    'la_liga': ['CCTV5', '咪咕体育'],
    'serie_a': ['咪咕体育'],
    'bundesliga': ['咪咕体育'],
    'ligue_1': ['咪咕体育'],
    'csl': ['腾讯体育', '咪咕体育'],
    'badminton': ['优酷体育', 'CCTV5+'],
    'tennis': ['优酷体育', 'CCTV5+'],
    'lpl': ['B 站', '腾讯体育'],
    'lck': ['B 站'],
    'f1': ['CCTV5', '腾讯体育'],
    'khl': ['咪咕体育'],
    'nhl': ['腾讯体育'],
    'olympics': ['CCTV5', 'CCTV5+', '央视频']
}


def get_broadcast_platforms(sport_key: str, league_key: str = None) -> List[str]:
    """
    获取直播平台列表（v3.0.7 新增）
    
    Args:
        sport_key: 运动种类 key
        league_key: 联赛 key
    
    Returns:
        直播平台列表
    """
    # 优先使用联赛 key
    if league_key and league_key in BROADCAST_PLATFORMS:
        return BROADCAST_PLATFORMS[league_key]
    
    # 使用运动种类 key
    if sport_key in BROADCAST_PLATFORMS:
        return BROADCAST_PLATFORMS[sport_key]
    
    # 默认平台
    return ['待定']
```

---

### 7. 更新输出逻辑使用表格

**位置**: `sports_monitor.py` `query_today_matches()` 函数内

**示例 - 篮球输出部分**:
```python
# 原代码（约第 1000 行）
# for game in nba_games:
#     basketball_content.append(format_basketball_game(game, teams, 'high'))

# 改进后
nba_rows = []
for game in nba_games:
    formatted = format_basketball_game(game, teams, 'high' if any(t in game.get('home', '') or t in game.get('away', '') for t in teams) else 'medium')
    nba_rows.append([
        formatted['time'],
        formatted['matchup'],
        formatted['competition'],
        formatted['broadcast']
    ])

if nba_rows:
    basketball_content.append(render_ascii_table(
        headers=['时间', '对阵信息', '赛事名称', '直播平台'],
        rows=nba_rows,
        col_alignments=['center', 'left', 'left', 'center']
    ))
```

**示例 - 足球输出部分**:
```python
# 改进后
football_rows = []
for fixture in all_fixtures:
    formatted = format_football_fixture(fixture, teams)
    football_rows.append([
        formatted['time'],
        formatted['matchup'],
        formatted['competition'],
        formatted['broadcast'],
        formatted['status']
    ])

if football_rows:
    football_content.append(render_ascii_table(
        headers=['时间', '对阵信息', '赛事名称', '直播平台', '状态'],
        rows=football_rows,
        col_alignments=['center', 'left', 'left', 'center', 'center']
    ))
```

**示例 - 羽毛球输出部分**:
```python
# 改进后
badminton_rows = []
for match in badminton_data:
    formatted = format_badminton_match(match, players)
    badminton_rows.append([
        formatted['time'],
        formatted['matchup'],
        formatted['competition'],
        formatted['broadcast'],
        formatted['round']
    ])

if badminton_rows:
    result.append(render_ascii_table(
        headers=['时间', '对阵信息', '赛事名称', '直播平台', '轮次'],
        rows=badminton_rows,
        col_alignments=['center', 'left', 'left', 'center', 'center']
    ))
```

---

### 8. 添加 JSON 输出支持

**位置**: `sports_monitor.py` 主函数部分

```python
def output_json(data: dict) -> str:
    """
    输出 JSON 格式（v3.0.7 新增）
    
    Args:
        data: 赛事数据字典
    
    Returns:
        JSON 字符串
    """
    return json.dumps(data, ensure_ascii=False, indent=2)


# 在主函数中添加 --json 参数支持
parser.add_argument('--json', action='store_true', help='输出 JSON 格式')

# 在输出逻辑中
if args.json:
    print(output_json(all_sports_data))
else:
    print(query_today_matches(config, interests))
```

---

## 实施步骤

### 阶段 1: 基础函数（优先级：高）

1. ✅ 添加 `render_ascii_table()` 函数
2. ✅ 添加 `get_broadcast_platforms()` 函数
3. ✅ 添加 `BROADCAST_PLATFORMS` 映射表

**预计时间**: 30 分钟

### 阶段 2: 格式化函数重构（优先级：高）

1. ✅ 重构 `format_basketball_game()` 返回 dict
2. ✅ 重构 `format_football_fixture()` 返回 dict
3. ✅ 新增 `format_badminton_match()`
4. ✅ 新增 `format_esports_match()`

**预计时间**: 45 分钟

### 阶段 3: 输出逻辑更新（优先级：中）

1. ⏳ 更新篮球输出逻辑使用表格
2. ⏳ 更新足球输出逻辑使用表格
3. ⏳ 更新羽毛球输出逻辑使用表格
4. ⏳ 更新电竞输出逻辑使用表格

**预计时间**: 60 分钟

### 阶段 4: 测试与优化（优先级：中）

1. ⏳ 测试所有运动种类输出
2. ⏳ 验证直播平台显示
3. ⏳ 测试 JSON 输出
4. ⏳ 优化表格宽度自适应

**预计时间**: 30 分钟

---

## 测试用例

### 测试 1: 篮球表格输出

```bash
python3 sports_monitor.py --today 2>&1 | grep -A 20 "NBA"
```

**预期输出**:
```
┌──────────┬─────────────────────────────┬──────────────┬────────────┐
│  时间    │         对阵信息            │   赛事名称   │  直播平台  │
├──────────┼─────────────────────────────┼──────────────┼────────────┤
│ 08:00    │ Celtics @ Mavericks         │ NBA 常规赛   │ CCTV5      │
└──────────┴─────────────────────────────┴──────────────┴────────────┘
```

### 测试 2: 足球表格输出

```bash
python3 sports_monitor.py --today 2>&1 | grep -A 20 "欧洲五大联赛"
```

**预期输出**:
```
┌──────────┬─────────────────────────────┬──────────────┬────────────┬──────────────┐
│  时间    │         对阵信息            │   赛事名称   │  直播平台  │    状态      │
├──────────┼─────────────────────────────┼──────────────┼────────────┼──────────────┤
│ 20:00    │ Man City vs Liverpool       │ 英超第 29 轮  │ 咪咕体育   │ 未开始      │
└──────────┴─────────────────────────────┴──────────────┴────────────┴──────────────┘
```

### 测试 3: JSON 输出

```bash
python3 sports_monitor.py --today --json | python3 -m json.tool | head -30
```

**预期输出**: 有效的 JSON 格式

---

## 注意事项

1. **向后兼容**: 保留原有函数签名，新增函数使用新名称
2. **性能考虑**: 表格渲染不应显著增加响应时间
3. **平台适配**: 考虑 Discord/微信等平台的字符限制
4. **错误处理**: 表格渲染失败时降级为纯文本输出

---

## 参考文档

- `OUTPUT_FORMAT.md` - 详细输出格式规范
- `DEVELOPMENT.md` - 开发文档
- `sports_monitor.py` - 当前源代码

---

*创建日期：2026-03-11*  
*版本：v3.0.7*
