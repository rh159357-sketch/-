import React, { useMemo, useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle, DialogTrigger } from "@/components/ui/dialog";
import { Label } from "@/components/ui/label";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { Calendar } from "@/components/ui/calendar";
import { CalendarIcon, ClipboardListIcon, FileTextIcon, PlusCircle, QrCode, RefreshCw, SearchIcon, ShieldCheck, WrenchIcon } from "lucide-react";
import { format } from "date-fns";

// -----------------------------------------------
// 복지관 내부용: 보장구(휠체어 등) 대여·수리 관리 MVP
// - 문성배 사업(보장구 대여·수리 지원) 흐름 반영
// - 단일 파일 React 컴포넌트 (shadcn/ui + Tailwind)
// -----------------------------------------------

// 읍·면 코드 (고창군)
const EUPMYEON = ["고창","고수","공음","대산","무장","부안","상하","성내","성송","신림","심원","아산","해리","흥덕"] as const;

// 장비/대여/점검 상태
const DEVICE_STATUS = ["AVAILABLE","RENTED","MAINTAIN","REPAIR","INACTIVE"] as const;
const RENTAL_STATUS = ["RESERVED","RENTED","RETURNED","OVERDUE","CANCELLED"] as const;

// 타입 정의
type EupMyeon = typeof EUPMYEON[number];
type DeviceStatus = typeof DEVICE_STATUS[number];
type RentalStatus = typeof RENTAL_STATUS[number];

interface Device {
  id: string;
  serial: string;
  type: "MANUAL" | "POWER" | "SCOOTER";
  width_mm?: number;
  max_load_kg?: number;
  status: DeviceStatus;
  last_sanitized_at?: string;
  last_inspected_at?: string;
}

interface Person {
  id: string;
  name: string;
  birth: string; // YYYY-MM-DD
  phone: string;
  eup_myeon: EupMyeon;
  notes?: string;
}

interface Rental {
  id: string;
  person_id: string;
  device_id: string;
  start_at: string; // ISO
  end_at: string;   // ISO
  status: RentalStatus;
  checkin_at?: string;
  checkout_at?: string;
  deposit_amount?: number;
  pickup_code?: string;
}

interface Maintenance {
  id: string;
  device_id: string;
  kind: "SANITIZE" | "INSPECT" | "REPAIR";
  issue?: "TIRE" | "BRAKE" | "FOOTREST" | "ARMREST" | "BELT" | "FRAME" | "BATTERY" | "OTHER";
  note?: string;
  material_cost?: number;
  done_at?: string;
}

// -----------------------------------------------
// 목업 데이터 (실서비스에선 API 연동)
// -----------------------------------------------
const mockDevices: Device[] = [
  { id: "D-001", serial: "WCH-0001", type: "MANUAL", width_mm: 450, max_load_kg: 120, status: "AVAILABLE", last_sanitized_at: "2025-10-10", last_inspected_at: "2025-09-20" },
  { id: "D-002", serial: "WCH-0002", type: "POWER", width_mm: 500, max_load_kg: 140, status: "RENTED", last_sanitized_at: "2025-10-11", last_inspected_at: "2025-09-15" },
  { id: "D-003", serial: "SCO-0003", type: "SCOOTER", width_mm: 520, max_load_kg: 150, status: "MAINTAIN" },
  { id: "D-004", serial: "WCH-0004", type: "MANUAL", width_mm: 430, max_load_kg: 110, status: "REPAIR" },
];

const mockPeople: Person[] = [
  { id: "P-001", name: "김고창", birth: "1965-04-03", phone: "010-1111-2222", eup_myeon: "고창" },
  { id: "P-002", name: "이흥덕", birth: "1959-12-09", phone: "010-3333-4444", eup_myeon: "흥덕" },
  { id: "P-003", name: "박성내", birth: "1978-07-21", phone: "010-5555-6666", eup_myeon: "성내" },
];

const mockRentals: Rental[] = [
  { id: "R-1001", person_id: "P-001", device_id: "D-002", start_at: "2025-10-12T10:00:00", end_at: "2025-10-15T18:00:00", status: "RENTED", checkin_at: "2025-10-12T10:10:00", deposit_amount: 0, pickup_code: "381029" },
];

const mockMaint: Maintenance[] = [
  { id: "M-2001", device_id: "D-003", kind: "SANITIZE" },
  { id: "M-2002", device_id: "D-004", kind: "REPAIR", issue: "TIRE", note: "앞바퀴 교체 필요", material_cost: 0 },
];

// 유틸
const statusBadge = (s: DeviceStatus) => {
  const map: Record<DeviceStatus, string> = {
    AVAILABLE: "bg-emerald-100 text-emerald-700",
    RENTED: "bg-blue-100 text-blue-700",
    MAINTAIN: "bg-amber-100 text-amber-700",
    REPAIR: "bg-rose-100 text-rose-700",
    INACTIVE: "bg-gray-100 text-gray-600",
  };
  return <Badge className={`rounded-full ${map[s]}`}>{s}</Badge>;
};

function useMockStore() {
  const [devices, setDevices] = useState<Device[]>(mockDevices);
  const [people] = useState<Person[]>(mockPeople);
  const [rentals, setRentals] = useState<Rental[]>(mockRentals);
  const [maint, setMaint] = useState<Maintenance[]>(mockMaint);

  // 예약 생성 (겹침/상태 검사 최소 구현)
  function createReservation(person_id: string, device_id: string, start_at: Date, end_at: Date) {
    const d = devices.find((x) => x.id === device_id);
    if (!d) throw new Error("장비 없음");
    if (d.status !== "AVAILABLE") throw new Error("장비 가용 상태가 아닙니다.");

    const overlap = rentals.some((r) => r.device_id === device_id && ["RESERVED","RENTED"].includes(r.status) && new Date(r.start_at) < end_at && new Date(r.end_at) > start_at);
    if (overlap) throw new Error("동일 기간에 예약/대여가 존재합니다.");

    const id = `R-${Math.floor(Math.random() * 9000 + 1000)}`;
    const pickup_code = String(Math.floor(100000 + Math.random() * 900000));

    setRentals((prev) => [
      ...prev,
      { id, person_id, device_id, start_at: start_at.toISOString(), end_at: end_at.toISOString(), status: "RESERVED", pickup_code },
    ]);
    return { id, pickup_code };
  }

  function checkinRental(rental_id: string, code: string) {
    setRentals((prev) => prev.map((r) => {
      if (r.id === rental_id) {
        if (r.status !== "RESERVED") throw new Error("예약 상태가 아닙니다.");
        if (r.pickup_code !== code) throw new Error("코드가 일치하지 않습니다.");
        // 장비 상태 전환
        setDevices((dv) => dv.map((d) => (d.id === r.device_id ? { ...d, status: "RENTED" } : d)));
        return { ...r, status: "RENTED", checkin_at: new Date().toISOString() };
      }
      return r;
    }));
  }

  function checkoutRental(rental_id: string, damageNote?: string) {
    // 반납 → 장비 MAINTAIN, 소독/점검 작업 생성
    const target = rentals.find((r) => r.id === rental_id);
    if (!target) throw new Error("대여 내역 없음");
    if (!(["RENTED","OVERDUE"].includes(target.status))) throw new Error("대여 중 상태가 아닙니다.");

    setRentals((prev) => prev.map((r) => (r.id === rental_id ? { ...r, status: "RETURNED", checkout_at: new Date().toISOString() } : r)));
    setDevices((prev) => prev.map((d) => (d.id === target.device_id ? { ...d, status: "MAINTAIN" } : d)));
    setMaint((prev) => [
      ...prev,
      { id: `M-${Math.floor(Math.random() * 9000 + 1000)}`, device_id: target.device_id, kind: "SANITIZE", note: damageNote },
      { id: `M-${Math.floor(Math.random() * 9000 + 1000)}`, device_id: target.device_id, kind: "INSPECT" },
    ]);
  }

  function completeMaintenance(mid: string, payload: Partial<Maintenance>) {
    setMaint((prev) => prev.map((m) => (m.id === mid ? { ...m, ...payload, done_at: new Date().toISOString() } : m)));

    // 해당 장비의 미완료 작업 잔존 여부 확인 → 없으면 AVAILABLE 전환
    const m = maint.find((x) => x.id === mid);
    if (!m) return;
    const deviceId = m.device_id;
    const hasRemain = maint.some((x) => x.device_id === deviceId && !x.done_at && x.id !== mid);
    if (!hasRemain) {
      setDevices((prev) => prev.map((d) => (d.id === deviceId ? { ...d, status: "AVAILABLE", last_sanitized_at: format(new Date(), "yyyy-MM-dd"), last_inspected_at: format(new Date(), "yyyy-MM-dd") } : d)));
    }
  }

  function reportDamage(device_id: string, issue: Maintenance["issue"], note?: string) {
    setDevices((prev) => prev.map((d) => (d.id === device_id ? { ...d, status: "REPAIR" } : d)));
    setMaint((prev) => [...prev, { id: `M-${Math.floor(Math.random() * 9000 + 1000)}`, device_id, kind: "REPAIR", issue, note }]);
  }

  return { devices, people, rentals, maint, createReservation, checkinRental, checkoutRental, completeMaintenance, reportDamage };
}

// 공용 헤더
function HeaderBar() {
  return (
    <div className="flex items-center justify-between mb-6">
      <div className="flex items-center gap-2">
        <ShieldCheck className="h-6 w-6" />
        <h1 className="text-2xl font-bold">보장구 대여·수리지원 (복지관 내부)</h1>
      </div>
      <div className="text-sm text-muted-foreground">문성배 사업 기준 · 개인정보 최소열람</div>
    </div>
  );
}

// 대시보드
function Dashboard({ devices, rentals, maint }: { devices: Device[]; rentals: Rental[]; maint: Maintenance[] }) {
  const today = format(new Date(), "yyyy-MM-dd");
  const available = devices.filter((d) => d.status === "AVAILABLE").length;
  const rented = devices.filter((d) => d.status === "RENTED").length;
  const maintain = devices.filter((d) => d.status === "MAINTAIN").length;
  const repairing = devices.filter((d) => d.status === "REPAIR").length;
  const dueToday = rentals.filter((r) => r.status === "RENTED" && r.end_at.slice(0, 10) === today);

  return (
    <div className="grid md:grid-cols-4 gap-4">
      <Card className="col-span-1"><CardHeader><CardTitle>가용</CardTitle><CardDescription>AVAILABLE</CardDescription></CardHeader><CardContent><p className="text-4xl font-bold">{available}</p></CardContent></Card>
      <Card className="col-span-1"><CardHeader><CardTitle>대여중</CardTitle><CardDescription>RENTED</CardDescription></CardHeader><CardContent><p className="text-4xl font-bold">{rented}</p></CardContent></Card>
      <Card className="col-span-1"><CardHeader><CardTitle>점검/소독 대기</CardTitle><CardDescription>MAINTAIN</CardDescription></CardHeader><CardContent><p className="text-4xl font-bold">{maintain}</p></CardContent></Card>
      <Card className="col-span-1"><CardHeader><CardTitle>수리중</CardTitle><CardDescription>REPAIR</CardDescription></CardHeader><CardContent><p className="text-4xl font-bold">{repairing}</p></CardContent></Card>

      <Card className="col-span-4"><CardHeader><CardTitle>오늘 반납 예정</CardTitle><CardDescription>{today}</CardDescription></CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>대여ID</TableHead>
                <TableHead>대상자</TableHead>
                <TableHead>장비</TableHead>
                <TableHead>종료</TableHead>
                <TableHead>상태</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {dueToday.length === 0 && (
                <TableRow><TableCell colSpan={5} className="text-center text-muted-foreground">해당없음</TableCell></TableRow>
              )}
              {dueToday.map((r) => (
                <TableRow key={r.id}>
                  <TableCell>{r.id}</TableCell>
                  <TableCell>{r.person_id}</TableCell>
                  <TableCell>{r.device_id}</TableCell>
                  <TableCell>{format(new Date(r.end_at), "MM/dd HH:mm")}</TableCell>
                  <TableCell>{r.status}</TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}

// 장비 탭
function DevicesTab({ devices, onReportDamage }: { devices: Device[]; onReportDamage: (id: string, issue: Maintenance["issue"], note?: string) => void }) {
  const [q, setQ] = useState("");
  const filtered = useMemo(() => devices.filter((d) => [d.id, d.serial, d.type, d.status].join(" ").toLowerCase().includes(q.toLowerCase())), [devices, q]);
  const [issue, setIssue] = useState<Maintenance["issue"]>("OTHER");
  const [note, setNote] = useState("");
  const [target, setTarget] = useState<string | null>(null);

  return (
    <div className="space-y-4">
      <div className="flex gap-2 items-center">
        <div className="relative w-full md:w-1/2">
          <SearchIcon className="absolute left-2 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
          <Input value={q} onChange={(e) => setQ(e.target.value)} placeholder="장비 ID/시리얼/상태 검색" className="pl-8" />
        </div>
      </div>
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>장비ID</TableHead>
            <TableHead>시리얼</TableHead>
            <TableHead>유형</TableHead>
            <TableHead>좌폭/하중</TableHead>
            <TableHead>상태</TableHead>
            <TableHead className="text-right">작업</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {filtered.map((d) => (
            <TableRow key={d.id}>
              <TableCell>{d.id}</TableCell>
              <TableCell>{d.serial}</TableCell>
              <TableCell>{d.type}</TableCell>
              <TableCell>{d.width_mm ?? "-"}/{d.max_load_kg ?? "-"}</TableCell>
              <TableCell>{statusBadge(d.status)}</TableCell>
              <TableCell className="text-right">
                <Dialog>
                  <DialogTrigger asChild>
                    <Button variant="outline" size="sm" onClick={() => setTarget(d.id)} className="gap-1"><WrenchIcon className="h-4 w-4"/>파손 신고</Button>
                  </DialogTrigger>
                  <DialogContent>
                    <DialogHeader>
                      <DialogTitle>파손 신고 · {target}</DialogTitle>
                      <DialogDescription>수리 요청을 생성합니다.</DialogDescription>
                    </DialogHeader>
                    <div className="grid gap-3">
                      <Label>이슈</Label>
                      <Select value={issue} onValueChange={(v: any) => setIssue(v)}>
                        <SelectTrigger><SelectValue placeholder="선택"/></SelectTrigger>
                        <SelectContent>
                          {(["TIRE","BRAKE","FOOTREST","ARMREST","BELT","FRAME","BATTERY","OTHER"] as const).map((k) => (
                            <SelectItem key={k} value={k}>{k}</SelectItem>
                          ))}
                        </SelectContent>
                      </Select>
                      <Label>메모</Label>
                      <Textarea value={note} onChange={(e) => setNote(e.target.value)} placeholder="증상/상황"/>
                    </div>
                    <DialogFooter>
                      <Button onClick={() => { if (target) onReportDamage(target, issue, note); }}>등록</Button>
                    </DialogFooter>
                  </DialogContent>
                </Dialog>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  );
}

// 대여 탭 (예약/체크인/반납)
function RentalsTab({ people, devices, rentals, onReserve, onCheckin, onCheckout }: {
  people: Person[];
  devices: Device[];
  rentals: Rental[];
  onReserve: (person_id: string, device_id: string, start: Date, end: Date) => { id: string; pickup_code: string };
  onCheckin: (rental_id: string, code: string) => void;
  onCheckout: (rental_id: string, note?: string) => void;
}) {
  const [person, setPerson] = useState<string>("");
  const [device, setDevice] = useState<string>("");
  const [start, setStart] = useState<Date | undefined>(new Date());
  const [end, setEnd] = useState<Date | undefined>(new Date(Date.now() + 24*60*60*1000));
  const [pickup, setPickup] = useState<string>("");
  const availableDevices = devices.filter((d) => d.status === "AVAILABLE");

  const [filterRegion, setFilterRegion] = useState<EupMyeon | "">("");
  const list = useMemo(() => rentals.filter(r => filterRegion ? (people.find(p=>p.id===r.person_id)?.eup_myeon===filterRegion) : true), [rentals, filterRegion, people]);

  return (
    <div className="grid lg:grid-cols-3 gap-4">
      <Card className="lg:col-span-1">
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><PlusCircle className="h-5 w-5"/>예약 생성</CardTitle>
          <CardDescription>겹침·상태 자동 검증</CardDescription>
        </CardHeader>
        <CardContent className="space-y-3">
          <Label>대상자</Label>
          <Select value={person} onValueChange={setPerson}>
            <SelectTrigger><SelectValue placeholder="선택"/></SelectTrigger>
            <SelectContent>
              {people.map((p) => (
                <SelectItem key={p.id} value={p.id}>{p.name} · {p.eup_myeon}</SelectItem>
              ))}
            </SelectContent>
          </Select>

          <Label>장비</Label>
          <Select value={device} onValueChange={setDevice}>
            <SelectTrigger><SelectValue placeholder="AVAILABLE 목록"/></SelectTrigger>
            <SelectContent>
              {availableDevices.map((d) => (
                <SelectItem key={d.id} value={d.id}>{d.id} · {d.type} · {d.serial}</SelectItem>
              ))}
            </SelectContent>
          </Select>

          <Label>기간</Label>
          <div className="grid grid-cols-2 gap-2">
            <Popover>
              <PopoverTrigger asChild>
                <Button variant="outline" className="justify-start"><CalendarIcon className="h-4 w-4 mr-2"/>{start ? format(start, "MM/dd HH:mm") : "시작"}</Button>
              </PopoverTrigger>
              <PopoverContent className="p-2">
                <Calendar mode="single" selected={start} onSelect={(d:any)=>setStart(d)} initialFocus />
              </PopoverContent>
            </Popover>
            <Popover>
              <PopoverTrigger asChild>
                <Button variant="outline" className="justify-start"><CalendarIcon className="h-4 w-4 mr-2"/>{end ? format(end, "MM/dd HH:mm") : "종료"}</Button>
              </PopoverTrigger>
              <PopoverContent className="p-2">
                <Calendar mode="single" selected={end} onSelect={(d:any)=>setEnd(d)} initialFocus />
              </PopoverContent>
            </Popover>
          </div>

          <Button className="w-full" onClick={() => {
            if (!person || !device || !start || !end) return alert("입력 누락");
            try {
              const res = onReserve(person, device, start, end);
              alert(`예약 완료! ID: ${res.id}, 픽업코드: ${res.pickup_code}`);
            } catch (e:any) { alert(e.message); }
          }}>예약 생성</Button>
        </CardContent>
      </Card>

      <Card className="lg:col-span-2">
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><ClipboardListIcon className="h-5 w-5"/>대여 목록</CardTitle>
          <CardDescription>체크인·반납 처리</CardDescription>
        </CardHeader>
        <CardContent>
          <div className="flex items-center gap-2 mb-3">
            <Select value={filterRegion} onValueChange={(v:any)=>setFilterRegion(v)}>
              <SelectTrigger className="w-[200px]"><SelectValue placeholder="읍·면 필터"/></SelectTrigger>
              <SelectContent>
                <SelectItem value="">전체</SelectItem>
                {EUPMYEON.map((e)=>(<SelectItem key={e} value={e}>{e}</SelectItem>))}
              </SelectContent>
            </Select>
          </div>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>ID</TableHead>
                <TableHead>대상자</TableHead>
                <TableHead>읍·면</TableHead>
                <TableHead>장비</TableHead>
                <TableHead>기간</TableHead>
                <TableHead>상태</TableHead>
                <TableHead className="text-right">작업</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {list.map((r)=>(
                <TableRow key={r.id}>
                  <TableCell>{r.id}</TableCell>
                  <TableCell>{people.find(p=>p.id===r.person_id)?.name}</TableCell>
                  <TableCell>{people.find(p=>p.id===r.person_id)?.eup_myeon}</TableCell>
                  <TableCell>{r.device_id}</TableCell>
                  <TableCell>{format(new Date(r.start_at),"MM/dd HH:mm")}~{format(new Date(r.end_at),"MM/dd HH:mm")}</TableCell>
                  <TableCell>{r.status}</TableCell>
                  <TableCell className="text-right">
                    {r.status==="RESERVED" && (
                      <div className="flex gap-2 justify-end">
                        <Input placeholder="픽업코드" value={pickup} onChange={(e)=>setPickup(e.target.value)} className="w-[120px]"/>
                        <Button size="sm" onClick={()=>{ try{ onCheckin(r.id, pickup); setPickup(""); }catch(e:any){ alert(e.message);} }} className="gap-1"><QrCode className="h-4 w-4"/>체크인</Button>
                      </div>
                    )}
                    {r.status==="RENTED" && (
                      <Button variant="outline" size="sm" onClick={()=> onCheckout(r.id)} className="gap-1"><RefreshCw className="h-4 w-4"/>반납</Button>
                    )}
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}

// 점검/수리 탭
function MaintenanceTab({ maint, devices, onComplete }: { maint: Maintenance[]; devices: Device[]; onComplete: (mid: string, payload: Partial<Maintenance>) => void }) {
  const pending = maint.filter((m) => !m.done_at);
  const [cost, setCost] = useState<string>("0");
  const [note, setNote] = useState<string>("");

  return (
    <div className="grid md:grid-cols-2 gap-4">
      {pending.map((m) => (
        <Card key={m.id}>
          <CardHeader>
            <CardTitle className="flex items-center gap-2"><WrenchIcon className="h-5 w-5"/>작업 {m.id}</CardTitle>
            <CardDescription>{m.kind} · {devices.find(d=>d.id===m.device_id)?.serial}</CardDescription>
          </CardHeader>
          <CardContent className="space-y-3">
            {m.kind === "REPAIR" && (
              <div className="grid grid-cols-2 gap-2">
                <div>
                  <Label>재료비(원)</Label>
                  <Input type="number" value={cost} onChange={(e)=>setCost(e.target.value)} />
                </div>
                <div>
                  <Label>이슈</Label>
                  <Input value={m.issue ?? ""} disabled/>
                </div>
                <div className="col-span-2">
                  <Label>비고</Label>
                  <Textarea value={note} onChange={(e)=>setNote(e.target.value)} placeholder="부품/조치 내용"/>
                </div>
              </div>
            )}
            {m.kind !== "REPAIR" && (
              <div>
                <Label>체크리스트 메모</Label>
                <Textarea value={note} onChange={(e)=>setNote(e.target.value)} placeholder="소독/점검 결과"/>
              </div>
            )}
            <Button onClick={()=> onComplete(m.id, { note, material_cost: Number(cost||0) })}>완료</Button>
          </CardContent>
        </Card>
      ))}
      {pending.length===0 && (
        <Card className="md:col-span-2"><CardHeader><CardTitle>대기 작업 없음</CardTitle><CardDescription>모든 장비가 가용 상태일 수 있습니다.</CardDescription></CardHeader><CardContent/></Card>
      )}
    </div>
  );
}

// 보고서 탭 (간단 요약)
function ReportsTab({ rentals, people, devices }: { rentals: Rental[]; people: Person[]; devices: Device[] }) {
  const [from, setFrom] = useState<string>("");
  const [to, setTo] = useState<string>("");
  const [region, setRegion] = useState<string>("");

  const report = useMemo(() => {
    const f = from ? new Date(from) : null;
    const t = to ? new Date(to) : null;
    const rows = rentals.filter((r) => (!f || new Date(r.start_at) >= f) && (!t || new Date(r.end_at) <= t) && (!region || people.find(p=>p.id===r.person_id)?.eup_myeon===region));
    const total = rows.length;
    const byStatus = Object.groupBy(rows, (r)=>r.status);
    const util = devices.map((d)=>({ device: d.id, status: d.status }));
    return { total, byStatus, util, rows };
  }, [rentals, from, to, region, people, devices]);

  return (
    <div className="space-y-3">
      <div className="flex flex-wrap gap-2 items-end">
        <div>
          <Label>시작</Label>
          <Input type="date" value={from} onChange={(e)=>setFrom(e.target.value)} />
        </div>
        <div>
          <Label>종료</Label>
          <Input type="date" value={to} onChange={(e)=>setTo(e.target.value)} />
        </div>
        <div>
          <Label>읍·면</Label>
          <Select value={region} onValueChange={setRegion}>
            <SelectTrigger className="w-[180px]"><SelectValue placeholder="전체"/></SelectTrigger>
            <SelectContent>
              <SelectItem value="">전체</SelectItem>
              {EUPMYEON.map((e)=>(<SelectItem key={e} value={e}>{e}</SelectItem>))}
            </SelectContent>
          </Select>
        </div>
        <Button variant="outline" className="ml-auto gap-1"><FileTextIcon className="h-4 w-4"/>CSV 내보내기</Button>
      </div>

      <div className="grid md:grid-cols-3 gap-3">
        <Card><CardHeader><CardTitle>총 대여건</CardTitle></CardHeader><CardContent><p className="text-3xl font-bold">{report.total}</p></CardContent></Card>
        <Card><CardHeader><CardTitle>상태 분포</CardTitle></CardHeader><CardContent>
          <ul className="text-sm space-y-1">
            {Object.entries(report.byStatus||{}).map(([s, arr]) => (
              <li key={s} className="flex justify-between"><span>{s}</span><span className="font-semibold">{(arr as Rental[]).length}</span></li>
            ))}
          </ul>
        </CardContent></Card>
        <Card><CardHeader><CardTitle>장비 현황(요약)</CardTitle></CardHeader><CardContent>
          <div className="flex flex-wrap gap-2">
            {devices.map((d)=> (
              <div key={d.id} className="text-xs px-2 py-1 rounded-full border">{d.id}: {d.status}</div>
            ))}
          </div>
        </CardContent></Card>
      </div>

      <Card>
        <CardHeader><CardTitle>대여 내역</CardTitle></CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>ID</TableHead>
                <TableHead>대상자</TableHead>
                <TableHead>읍·면</TableHead>
                <TableHead>장비</TableHead>
                <TableHead>시작</TableHead>
                <TableHead>종료</TableHead>
                <TableHead>상태</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {report.rows.map((r)=>(
                <TableRow key={r.id}>
                  <TableCell>{r.id}</TableCell>
                  <TableCell>{people.find(p=>p.id===r.person_id)?.name}</TableCell>
                  <TableCell>{people.find(p=>p.id===r.person_id)?.eup_myeon}</TableCell>
                  <TableCell>{r.device_id}</TableCell>
                  <TableCell>{format(new Date(r.start_at),"yyyy-MM-dd HH:mm")}</TableCell>
                  <TableCell>{format(new Date(r.end_at),"yyyy-MM-dd HH:mm")}</TableCell>
                  <TableCell>{r.status}</TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}

export default function App() {
  const store = useMockStore();
  const [tab, setTab] = useState("dashboard");

  return (
    <div className="p-4 md:p-6 max-w-7xl mx-auto">
      <HeaderBar />
      <Tabs value={tab} onValueChange={setTab} className="space-y-4">
        <TabsList className="grid grid-cols-5 w-full">
          <TabsTrigger value="dashboard">대시보드</TabsTrigger>
          <TabsTrigger value="devices">장비</TabsTrigger>
          <TabsTrigger value="rentals">대여</TabsTrigger>
          <TabsTrigger value="maintenance">점검·수리</TabsTrigger>
          <TabsTrigger value="reports">보고서</TabsTrigger>
        </TabsList>
        <TabsContent value="dashboard">
          <Dashboard devices={store.devices} rentals={store.rentals} maint={store.maint} />
        </TabsContent>
        <TabsContent value="devices">
          <DevicesTab devices={store.devices} onReportDamage={store.reportDamage} />
        </TabsContent>
        <TabsContent value="rentals">
          <RentalsTab
            people={store.people}
            devices={store.devices}
            rentals={store.rentals}
            onReserve={store.createReservation}
            onCheckin={store.checkinRental}
            onCheckout={store.checkoutRental}
          />
        </TabsContent>
        <TabsContent value="maintenance">
          <MaintenanceTab maint={store.maint} devices={store.devices} onComplete={store.completeMaintenance} />
        </TabsContent>
        <TabsContent value="reports">
          <ReportsTab rentals={store.rentals} people={store.people} devices={store.devices} />
        </TabsContent>
      </Tabs>

      <div className="mt-6 text-xs text-muted-foreground">
        ※ 데모 데이터/로직 포함. 실제 배포 시 API 연동, 접근권한(역할별), 감사로그, 개인정보 보호설정 적용 필요.
      </div>
    </div>
  );
}
