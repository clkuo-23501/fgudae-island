/**
 * 佛光大學應用經濟學系 - 宜蘭學習島嶼共用權限模組 (auth.js)
 * 功能：管理訪客 / 在校生狀態、學號格式驗證、姓名去識別化遮罩與子頁面權限守衛
 */
(function (global) {
  'use strict';

  const SESSION_KEY = 'fgu_econ_island_session_v1';
  const ROLE_HINT_KEY = 'yilan_island_hub_role_b';

  // 經濟系試算表部分名單比對（供顯示年級與姓名提示）
  const KNOWN_STUDENTS = {
    '111175101': { grade: '111級', name: '羅子涵' },
    '112175101': { grade: '112級', name: '馬範懿' },
    '112175102': { grade: '112級', name: '劉昱辰' },
    '113175101': { grade: '113級', name: '邱宥睿' },
    '113175105': { grade: '113級', name: '陳品睿' },
    '114175001': { grade: '114級', name: '陳家佩' },
    '115175003': { grade: '115級', name: '陳亮宇' }
  };

  const IslandAuth = {
    // 取得目前工作階段身分（預設為 visitor）
    getSession() {
      try {
        const raw = sessionStorage.getItem(SESSION_KEY);
        if (raw) {
          const parsed = JSON.parse(raw);
          if (parsed && parsed.role === 'student' && parsed.last3) {
            return parsed;
          }
        }
      } catch (e) {}
      return { role: 'visitor', last3: '', grade: '', displayName: '' };
    },

    // 驗證學號並寫入同網域 sessionStorage（關閉分頁即自動清除）
    loginAsStudent(rawId) {
      const cleanId = String(rawId || '').trim().toUpperCase();
      if (!/^[A-Za-z0-9]{7,10}$/.test(cleanId)) {
        return { ok: false, error: '請輸入 7 到 10 碼英數字學號。' };
      }
      const profile = KNOWN_STUDENTS[cleanId] || null;
      const sessionData = {
        role: 'student',
        last3: cleanId.slice(-3),
        grade: profile ? profile.grade : '在校生',
        displayName: profile ? profile.name : ''
      };
      try {
        sessionStorage.setItem(SESSION_KEY, JSON.stringify(sessionData));
        localStorage.setItem(ROLE_HINT_KEY, 'student');
      } catch (e) {}
      return { ok: true, session: sessionData, fullId: cleanId };
    },

    // 切換為訪客模式
    switchToVisitor() {
      try {
        sessionStorage.removeItem(SESSION_KEY);
        localStorage.setItem(ROLE_HINT_KEY, 'visitor');
      } catch (e) {}
      return this.getSession();
    },

    isStudent() {
      return this.getSession().role === 'student';
    },

    // 依權限格式化姓名（訪客自動遮罩為「陳○○」）
    maskName(fullName) {
      if (!fullName) return '同學';
      if (this.isStudent()) return fullName;
      return fullName.charAt(0) + '○'.repeat(Math.max(1, fullName.length - 1));
    },

    // 子頁面權限守衛：若為專屬頁面（如職場副本島）且非在校生，自動導回主島或顯示提示
    guardRestrictedPage(options = {}) {
      if (this.isStudent()) return true;
      const redirectUrl = options.redirectUrl || 'index.html';
      if (options.autoRedirect) {
        window.location.replace(redirectUrl);
      }
      return false;
    }
  };

  global.IslandAuth = IslandAuth;
})(window);