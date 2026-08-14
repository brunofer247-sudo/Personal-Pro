[index.html](https://github.com/user-attachments/files/31050267/index.html)
[package.json](https://github.com/user-attachments/files/31050269/package.json)
[vite.config.js](https://github.com/user-attachments/files/31050271/vite.config.js)
[README.md](https://github.com/user-attachments/files/31050270/README.md)
import React, { useState, useEffect, useCallback } from "react";
import { Dumbbell, User, Plus, LogOut, Mail, Lock, AlertCircle } from "lucide-react";

// =================================================================
// CONFIGURAÇÃO — troque pelos valores do SEU projeto Supabase
// (Project Settings > API, no painel do Supabase)
// =================================================================
const CONFIG = {
  SUPABASE_URL: "https://parhyztbhyrhdqknhjwo.supabase.co",
  SUPABASE_ANON_KEY: "sb_publishable_GB0cpxroxWkaH6LDF6ZABw__MDMDyKY",
};

// ---------- design tokens (mesma identidade visual do CICLO) ----------
const T = {
  bg: "#F6F5F1", ink: "#22252A", inkSoft: "#5A5D63",
  card: "#FFFFFF", line: "#E1DFD6", accent: "#E4572E", accent2: "#3E5C76",
};

// =================================================================
// Camada de dados — chama a API REST do Supabase direto via fetch,
// sem depender do pacote @supabase/supabase-js (que não pode ser
// importado dentro de um artifact).
// =================================================================
function supaHeaders(session, extra = {}) {
  return {
    apikey: CONFIG.SUPABASE_ANON_KEY,
    Authorization: `Bearer ${session ? session.access_token : CONFIG.SUPABASE_ANON_KEY}`,
    "Content-Type": "application/json",
    ...extra,
  };
}

async function supaAuth(path, body) {
  const res = await fetch(`${CONFIG.SUPABASE_URL}/auth/v1/${path}`, {
    method: "POST",
    headers: { apikey: CONFIG.SUPABASE_ANON_KEY, "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const json = await res.json();
  if (!res.ok) throw new Error(json.error_description || json.msg || json.error || "Falha na autenticação");
  return json;
}

async function supaSelect(session, table, query = "") {
  const res = await fetch(`${CONFIG.SUPABASE_URL}/rest/v1/${table}?select=*${query}`, {
    headers: supaHeaders(session),
  });
  if (!res.ok) throw new Error(`Erro ao ler ${table} (${res.status})`);
  return res.json();
}

async function supaInsert(session, table, row) {
  const res = await fetch(`${CONFIG.SUPABASE_URL}/rest/v1/${table}`, {
    method: "POST",
    headers: supaHeaders(session, { Prefer: "return=representation" }),
    body: JSON.stringify(row),
  });
  if (!res.ok) throw new Error(`Erro ao gravar em ${table} (${res.status})`);
  const json = await res.json();
  return Array.isArray(json) ? json[0] : json;
}

async function supaDelete(session, table, id) {
  const res = await fetch(`${CONFIG.SUPABASE_URL}/rest/v1/${table}?id=eq.${id}`, {
    method: "DELETE",
    headers: supaHeaders(session),
  });
  if (!res.ok) throw new Error(`Erro ao excluir em ${table} (${res.status})`);
}

async function supaUpdate(session, table, id, patch) {
  const res = await fetch(`${CONFIG.SUPABASE_URL}/rest/v1/${table}?id=eq.${id}`, {
    method: "PATCH",
    headers: supaHeaders(session, { Prefer: "return=representation" }),
    body: JSON.stringify(patch),
  });
  if (!res.ok) throw new Error(`Erro ao atualizar em ${table} (${res.status})`);
  const json = await res.json();
  return Array.isArray(json) ? json[0] : json;
}

// ---------- UI atoms ----------
function Btn({ children, onClick, variant = "ghost", style, ...rest }) {
  const base = {
    fontFamily: "inherit", fontSize: 13, fontWeight: 600, padding: "8px 14px", borderRadius: 8,
    cursor: "pointer", display: "inline-flex", alignItems: "center", gap: 6,
    border: "1.5px solid transparent", transition: "all .12s ease",
  };
  const variants = {
    primary: { background: T.ink, color: "#fff" },
    accent: { background: T.accent, color: "#fff" },
    ghost: { background: "transparent", color: T.ink, borderColor: T.line },
    danger: { background: "transparent", color: "#A32D2D", borderColor: "#F0999522" },
  };
  return <button onClick={onClick} style={{ ...base, ...variants[variant], ...style }} {...rest}>{children}</button>;
}

// =================================================================
// Tela de login / cadastro
// =================================================================
function AuthScreen({ onAuthed }) {
  const [mode, setMode] = useState("login"); // 'login' | 'signup'
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState("");
  const configMissing = CONFIG.SUPABASE_URL.includes("SEU-PROJETO");

  async function submit() {
    setError("");
    if (!email || !password) return;
    setBusy(true);
    try {
      if (mode === "signup") {
        await supaAuth("signup", { email, password });
        setError("Conta criada! Verifique seu e-mail se a confirmação estiver ativada, depois faça login.");
        setMode("login");
      } else {
        const session = await supaAuth("token?grant_type=password", { email, password });
        onAuthed(session);
      }
    } catch (e) {
      setError(e.message);
    } finally {
      setBusy(false);
    }
  }

  return (
    <div style={{
      minHeight: 480, display: "flex", alignItems: "center", justifyContent: "center",
      background: T.bg, fontFamily: "'Inter', system-ui, sans-serif", borderRadius: 12, border: `1px solid ${T.line}`,
    }}>
      <div style={{ width: 320, background: T.card, border: `1px solid ${T.line}`, borderRadius: 14, padding: 26 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 4 }}>
          <Dumbbell size={20} color={T.accent} strokeWidth={2.5} />
          <span style={{ fontWeight: 800, fontSize: 17 }}>CICLO</span>
        </div>
        <div style={{ fontSize: 12, color: T.inkSoft, marginBottom: 20 }}>
          {mode === "login" ? "Entre com sua conta de personal." : "Crie sua conta de personal."}
        </div>

        {configMissing && (
          <div style={{ display: "flex", gap: 6, background: "#FBEAE3", color: "#993C1D", fontSize: 11.5, padding: 10, borderRadius: 8, marginBottom: 14 }}>
            <AlertCircle size={14} style={{ flexShrink: 0, marginTop: 1 }} />
            Configure SUPABASE_URL e SUPABASE_ANON_KEY no topo do arquivo antes de usar.
          </div>
        )}

        <label style={{ fontSize: 10.5, fontWeight: 700, color: T.inkSoft, textTransform: "uppercase" }}>E-mail</label>
        <div style={{ position: "relative", marginTop: 3, marginBottom: 12 }}>
          <Mail size={14} color={T.inkSoft} style={{ position: "absolute", left: 10, top: "50%", transform: "translateY(-50%)" }} />
          <input type="email" value={email} onChange={(e) => setEmail(e.target.value)}
            style={{ width: "100%", padding: "9px 10px 9px 32px", borderRadius: 8, border: `1px solid ${T.line}`, boxSizing: "border-box", fontSize: 13 }} />
        </div>

        <label style={{ fontSize: 10.5, fontWeight: 700, color: T.inkSoft, textTransform: "uppercase" }}>Senha</label>
        <div style={{ position: "relative", marginTop: 3, marginBottom: 16 }}>
          <Lock size={14} color={T.inkSoft} style={{ position: "absolute", left: 10, top: "50%", transform: "translateY(-50%)" }} />
          <input type="password" value={password} onChange={(e) => setPassword(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && submit()}
            style={{ width: "100%", padding: "9px 10px 9px 32px", borderRadius: 8, border: `1px solid ${T.line}`, boxSizing: "border-box", fontSize: 13 }} />
        </div>

        {error && <div style={{ fontSize: 11.5, color: "#A32D2D", marginBottom: 12 }}>{error}</div>}

        <Btn variant="accent" onClick={submit} style={{ width: "100%", justifyContent: "center", opacity: busy ? 0.6 : 1 }}>
          {busy ? "Aguarde..." : mode === "login" ? "Entrar" : "Criar conta"}
        </Btn>

        <div style={{ textAlign: "center", marginTop: 14, fontSize: 12 }}>
          {mode === "login" ? (
            <span>Não tem conta? <a href="#" onClick={(e) => { e.preventDefault(); setMode("signup"); setError(""); }} style={{ color: T.accent2, fontWeight: 700 }}>Criar agora</a></span>
          ) : (
            <span>Já tem conta? <a href="#" onClick={(e) => { e.preventDefault(); setMode("login"); setError(""); }} style={{ color: T.accent2, fontWeight: 700 }}>Entrar</a></span>
          )}
        </div>
      </div>
    </div>
  );
}

// =================================================================
// App principal (pós-login) — módulo de Alunos como padrão de referência
// =================================================================
function MainApp({ session, onLogout }) {
  const [students, setStudents] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");
  const [newName, setNewName] = useState("");
  const [newGoal, setNewGoal] = useState("");
  const [saving, setSaving] = useState(false);

  const refresh = useCallback(async () => {
    setLoading(true);
    setError("");
    try {
      const rows = await supaSelect(session, "students", "&order=created_at.desc");
      setStudents(rows);
    } catch (e) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  }, [session]);

  useEffect(() => { refresh(); }, [refresh]);

  async function addStudent() {
    if (!newName.trim()) return;
    setSaving(true);
    setError("");
    try {
      const row = await supaInsert(session, "students", {
        trainer_id: session.user.id,
        name: newName.trim(),
        goal: newGoal.trim(),
      });
      setStudents((prev) => [row, ...prev]);
      setNewName("");
      setNewGoal("");
    } catch (e) {
      setError(e.message);
    } finally {
      setSaving(false);
    }
  }

  async function removeStudent(id) {
    try {
      await supaDelete(session, "students", id);
      setStudents((prev) => prev.filter((s) => s.id !== id));
    } catch (e) {
      setError(e.message);
    }
  }

  return (
    <div style={{
      background: T.bg, color: T.ink, fontFamily: "'Inter', system-ui, sans-serif",
      borderRadius: 12, border: `1px solid ${T.line}`, minHeight: 480, padding: 24,
    }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 18 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <Dumbbell size={18} color={T.accent} strokeWidth={2.5} />
          <span style={{ fontWeight: 800, fontSize: 15 }}>CICLO</span>
          <span style={{ fontSize: 11.5, color: T.inkSoft, marginLeft: 6 }}>{session.user.email}</span>
        </div>
        <Btn onClick={onLogout}><LogOut size={13} /> Sair</Btn>
      </div>

      <div style={{ fontSize: 12, color: T.inkSoft, marginBottom: 16, background: "#EAF3DE", padding: "8px 12px", borderRadius: 8 }}>
        Conectado ao seu banco Supabase — os alunos abaixo são reais, gravados na nuvem e só visíveis pra você.
      </div>

      {error && (
        <div style={{ fontSize: 12, color: "#A32D2D", marginBottom: 14, background: "#FBEAE3", padding: "8px 12px", borderRadius: 8 }}>
          {error}
        </div>
      )}

      <div style={{ display: "flex", gap: 8, marginBottom: 16, flexWrap: "wrap" }}>
        <input placeholder="Nome do aluno" value={newName} onChange={(e) => setNewName(e.target.value)}
          style={{ flex: "1 1 160px", padding: "8px 10px", borderRadius: 8, border: `1px solid ${T.line}`, fontSize: 13 }} />
        <input placeholder="Objetivo (opcional)" value={newGoal} onChange={(e) => setNewGoal(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && addStudent()}
          style={{ flex: "1 1 160px", padding: "8px 10px", borderRadius: 8, border: `1px solid ${T.line}`, fontSize: 13 }} />
        <Btn variant="accent" onClick={addStudent} style={{ opacity: saving ? 0.6 : 1 }}>
          <Plus size={14} /> {saving ? "Salvando..." : "Adicionar"}
        </Btn>
      </div>

      {loading ? (
        <div style={{ fontSize: 13, color: T.inkSoft, textAlign: "center", marginTop: 30 }}>Carregando alunos...</div>
      ) : students.length === 0 ? (
        <div style={{ fontSize: 13, color: T.inkSoft, textAlign: "center", marginTop: 30 }}>Nenhum aluno cadastrado ainda.</div>
      ) : (
        students.map((s) => (
          <div key={s.id} style={{
            display: "flex", alignItems: "center", justifyContent: "space-between",
            background: T.card, border: `1px solid ${T.line}`, borderRadius: 8, padding: "10px 12px", marginBottom: 6,
          }}>
            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <User size={14} color={T.inkSoft} />
              <div>
                <div style={{ fontSize: 13.5, fontWeight: 600 }}>{s.name}</div>
                {s.goal && <div style={{ fontSize: 11.5, color: T.inkSoft }}>{s.goal}</div>}
              </div>
            </div>
            <Btn variant="danger" onClick={() => removeStudent(s.id)}>Excluir</Btn>
          </div>
        ))
      )}

      <div style={{ fontSize: 11, color: T.inkSoft, marginTop: 22, borderTop: `1px solid ${T.line}`, paddingTop: 12 }}>
        Este é o módulo de Alunos já ligado ao banco de dados de verdade — Periodização, Avaliação Física, Agenda e
        Financeiro seguem exatamente o mesmo padrão (mesma função supaSelect/supaInsert/supaDelete, outras tabelas)
        e entram na próxima etapa.
      </div>
    </div>
  );
}

// =================================================================
export default function CicloCloud() {
  const [session, setSession] = useState(null);

  if (!session) return <AuthScreen onAuthed={setSession} />;
  return <MainApp session={session} onLogout={() => setSession(null)} />;
}

import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <div style={{ maxWidth: 480, margin: "0 auto", padding: 12 }}>
      <App />
    </div>
  </React.StrictMode>
);
