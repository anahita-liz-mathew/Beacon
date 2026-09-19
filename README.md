# Beacon
Beacons app for people in crisis
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>BEACON - Offline Emergency Network</title>
  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- React & ReactDOM CDNs -->
  <script src="https://unpkg.com/react@18/umd/react.production.min.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js" crossorigin></script>
  <!-- Babel Compiler CDN -->
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-zinc-950 text-white min-h-screen flex items-center justify-center p-0 md:p-6 font-sans">
  <div id="root" class="w-full max-w-md h-screen md:h-[880px]"></div>

  <script type="text/babel">
    const { useState, useEffect, useRef } = React;

    function BeaconApp() {
      // Navigation & Layout State
      const [activeTab, setActiveTab] = useState('comms'); 
      const [batterySaver, setBatterySaver] = useState(false);
      const [toast, setToast] = useState(null);

      // Auto-Detect Current Hosted URL for Self-Referineing QR Code
      const getSelfUrl = () => {
        if (typeof window !== 'undefined' && window.location.protocol.startsWith('http')) {
          return window.location.href;
        }
        return 'https://your-beacon-app.netlify.app'; // Fallback example
      };

      const [qrTargetUrl, setQrTargetUrl] = useState(getSelfUrl());

      // Comms State
      const [isRecording, setIsRecording] = useState(false);
      const [broadcasts, setBroadcasts] = useState([
        { id: 1, sender: 'Doctor Mesh #04', text: 'Dr. Sarah active on mesh. Standing by for triage.', time: '1m ago', type: 'doctor' },
        { id: 2, sender: 'Node #8812', text: 'Clean drinking water available at Sector 3 center.', time: '4m ago', type: 'info' }
      ]);

      // Radar & Hazards State
      const [nodes, setNodes] = useState([
        { id: 'Dr. Sarah (Field MD)', dist: '60m', signal: 'Strong', hops: 1, status: 'Medical Personnel' },
        { id: 'Node #8812', dist: '45m', signal: 'Strong', hops: 1, status: 'Active' },
        { id: 'Rescue Hub Alpha', dist: '310m', signal: 'Weak', hops: 3, status: 'Base Station' }
      ]);
      const [hazards, setHazards] = useState([
        { id: 1, text: 'Flooded road at 2nd St bridge', severity: 'High', time: '10m ago' },
        { id: 2, text: 'Downed power line on Elm Ave', severity: 'Medium', time: '20m ago' }
      ]);
      const [newHazard, setNewHazard] = useState('');

      // Siren Audio State
      const [isSirenActive, setIsSirenActive] = useState(false);
      const [sirenTone, setSirenTone] = useState('whistle');
      const audioCtxRef = useRef(null);
      const oscRef = useRef(null);

      // Box Breathing Exercise State
      const [breathingPhase, setBreathingPhase] = useState('Breathe In');

      // GPS State
      const [gpsCoords, setGpsCoords] = useState({ lat: '37.7749 N', long: '122.4194 W' });
      const [activeFirstAid, setActiveFirstAid] = useState(null);

      // Toast Notification Utility
      const showToast = (msg, type = 'info') => {
        setToast({ msg, type });
        setTimeout(() => setToast(null), 3500);
      };

      // Real Time GPS Fetching
      useEffect(() => {
        if ("geolocation" in navigator) {
          navigator.geolocation.getCurrentPosition(
            (pos) => {
              setGpsCoords({
                lat: `${pos.coords.latitude.toFixed(4)}° N`,
                long: `${pos.coords.longitude.toFixed(4)}° W`
              });
            },
            () => { /* Keep default coords if offline */ }
          );
        }
      }, []);

      // Interactive Box Breathing Loop
      useEffect(() => {
        const phases = ['Breathe In (4s)', 'Hold Breath (4s)', 'Breathe Out (4s)', 'Hold Breath (4s)'];
        let step = 0;
        const interval = setInterval(() => {
          step = (step + 1) % 4;
          setBreathingPhase(phases[step]);
        }, 4000);
        return () => clearInterval(interval);
      }, []);

      // Web Audio Siren Synthesizer
      const stopAudioSiren = () => {
        if (oscRef.current) {
          try { oscRef.current.stop(); } catch(e){}
          oscRef.current.disconnect();
          oscRef.current = null;
        }
        if (audioCtxRef.current) {
          try { audioCtxRef.current.close(); } catch(e){}
          audioCtxRef.current = null;
        }
      };

      const startAudioSiren = (toneType) => {
        stopAudioSiren();
        const AudioContext = window.AudioContext || window.webkitAudioContext;
        if (!AudioContext) return;

        const ctx = new AudioContext();
        audioCtxRef.current = ctx;
        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        oscRef.current = osc;

        osc.connect(gain);
        gain.connect(ctx.destination);

        if (toneType === 'whistle') {
          osc.type = 'sine';
          osc.frequency.setValueAtTime(1200, ctx.currentTime);
          gain.gain.setValueAtTime(0.8, ctx.currentTime);
        } else if (toneType === 'siren') {
          osc.type = 'sawtooth';
          osc.frequency.setValueAtTime(600, ctx.currentTime);
          osc.frequency.linearRampToValueAtTime(1200, ctx.currentTime + 0.5);
          gain.gain.setValueAtTime(0.7, ctx.currentTime);
        } else if (toneType === 'dogWhistle') {
          osc.type = 'sine';
          osc.frequency.setValueAtTime(3800, ctx.currentTime);
          gain.gain.setValueAtTime(0.9, ctx.currentTime);
        }

        osc.start();
      };

      const toggleSiren = () => {
        if (!isSirenActive) {
          setIsSirenActive(true);
          startAudioSiren(sirenTone);
          showToast(`?? ${sirenTone.toUpperCase()} Siren Active!`, 'alert');
        } else {
          setIsSirenActive(false);
          stopAudioSiren();
          showToast('?? Siren Silenced', 'info');
        }
      };

      // SOS Trigger
      const handleSOS = () => {
        const newBroadcast = {
          id: Date.now(),
          sender: 'YOU (SOS)',
          text: `?? CRITICAL SOS BROADCAST - Location: ${gpsCoords.lat}, ${gpsCoords.long}`,
          time: 'Just now',
          type: 'alert'
        };
        setBroadcasts([newBroadcast, ...broadcasts]);
        showToast('?? CRITICAL SOS BROADCAST SENT TO ALL NEARBY NODES', 'alert');
      };

      // Request Medical Consultation
      const handleDoctorConsultRequest = () => {
        const docMsg = {
          id: Date.now(),
          sender: 'YOU (Medical Consult)',
          text: `?? REQUESTING FIELD DOCTOR / TRIAGE ASSISTANCE at ${gpsCoords.lat}, ${gpsCoords.long}`,
          time: 'Just now',
          type: 'doctor'
        };
        setBroadcasts([docMsg, ...broadcasts]);
        showToast('?? Priority Doctor Alert Transmitted across Mesh Network', 'success');
      };

      // Voice Note Simulator
      const handleVoiceNote = () => {
        if (!isRecording) {
          setIsRecording(true);
          showToast('??? Recording Voice Note...', 'info');
          setTimeout(() => {
            setIsRecording(false);
            const mockText = "Water supply low at Sector 2. 2 adults safe.";
            setBroadcasts(prev => [{
              id: Date.now(),
              sender: 'YOU (Voice Note)',
              text: `??? "${mockText}"`,
              time: 'Just now',
              type: 'info'
            }, ...prev]);
            showToast('? Voice Note Relayed to Mesh', 'success');
          }, 3000);
        }
      };

      return (
        <div className={`w-full h-full ${batterySaver ? 'bg-black' : 'bg-zinc-900'} text-zinc-100 flex flex-col justify-between shadow-2xl md:rounded-[40px] overflow-hidden border border-zinc-800 relative select-none`}>
          
          {/* Top Notification Toast */}
          {toast && (
            <div className={`absolute top-4 left-4 right-4 z-50 p-3.5 rounded-xl shadow-2xl text-xs font-bold flex items-center justify-between border backdrop-blur-md animate-bounce ${
              toast.type === 'alert' ? 'bg-red-950/95 border-red-500 text-red-200' :
              toast.type === 'success' ? 'bg-emerald-950/95 border-emerald-500 text-emerald-200' :
              'bg-zinc-800/95 border-zinc-600 text-zinc-100'
            }`}>
              <span className="flex items-center gap-2">{toast.msg}</span>
              <button onClick={() => setToast(null)} className="ml-2 text-base opacity-70 hover:opacity-100">?</button>
            </div>
          )}

          {/* Siren Strobe Overlay */}
          {isSirenActive && (
            <div className="absolute inset-0 z-40 bg-red-600/30 animate-pulse pointer-events-none"></div>
          )}

          {/* App Header */}
          <header className="p-4 border-b border-zinc-800/80 flex justify-between items-center bg-zinc-950/80 backdrop-blur-md">
            <div>
              <h1 className="text-lg font-black tracking-wider text-red-500 flex items-center gap-1.5">
                <span className="inline-block w-2.5 h-2.5 rounded-full bg-red-500 animate-ping"></span>
                BEACON
              </h1>
              <p className="text-[10px] text-zinc-400 font-mono">Mesh Node: #9901 • Offline Network Active</p>
            </div>
            <button 
              onClick={() => {
                setBatterySaver(!batterySaver);
                showToast(batterySaver ? 'Standard Mode' : 'OLED Battery Saver (True Black)', 'info');
              }}
              className="px-3 py-1 text-[11px] font-mono bg-zinc-800 hover:bg-zinc-700 rounded-full border border-zinc-700 text-zinc-300 transition">
              {batterySaver ? '?? Dark Max' : '?? Standard'}
            </button>
          </header>

          {/* Main Workspace */}
          <main className="flex-1 overflow-y-auto p-4 space-y-5">
            
            {/* TAB 1: COMMS & SOS */}
            {activeTab === 'comms' && (
              <div className="space-y-5">
                {/* Panic SOS Button */}
                <div className="flex flex-col items-center justify-center my-1">
                  <button 
                    onClick={handleSOS}
                    className="w-40 h-40 rounded-full bg-gradient-to-br from-red-600 via-red-700 to-red-900 hover:scale-105 active:scale-95 shadow-[0_0_50px_rgba(220,38,38,0.4)] transition-all flex flex-col items-center justify-center border-4 border-red-500/40">
                    <span className="text-3xl font-black tracking-widest text-white">SOS</span>
                    <span className="text-[9px] tracking-widest text-red-200 mt-1 uppercase font-mono">One-Tap Broadcast</span>
                  </button>
                </div>

                {/* Request Doctor On Mesh Button */}
                <button 
                  onClick={handleDoctorConsultRequest}
                  className="w-full py-3 rounded-2xl bg-emerald-950/60 hover:bg-emerald-900/60 border border-emerald-600/60 text-emerald-200 font-bold text-xs flex items-center justify-center gap-2 transition font-mono">
                  ?? REQUEST FIELD DOCTOR / MED-CONSULT
                </button>

                {/* Offline Walkie-Talkie */}
                <div className="bg-zinc-950/50 p-3.5 rounded-2xl border border-zinc-800 space-y-2">
                  <div className="flex justify-between items-center">
                    <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Offline Walkie-Talkie</h3>
                    <span className="text-[10px] bg-zinc-800 text-zinc-400 px-2 py-0.5 rounded font-mono">P2P Audio</span>
                  </div>
                  <button 
                    onClick={handleVoiceNote}
                    className={`w-full py-2.5 rounded-xl font-bold text-xs tracking-wider flex items-center justify-center gap-2 transition ${
                      isRecording ? 'bg-red-600 animate-pulse text-white' : 'bg-zinc-800 hover:bg-zinc-700 text-zinc-100 border border-zinc-700'
                    }`}>
                    {isRecording ? '?? Recording... Release to Send' : '??? Hold / Tap to Send Voice Note'}
                  </button>
                </div>

                {/* Quick Status Options */}
                <div className="space-y-2">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Quick Status Broadcast</h3>
                  <div className="grid grid-cols-2 gap-2">
                    {[
                      { msg: 'I am Safe ?' },
                      { msg: 'Need Water/Food ??' },
                      { msg: 'Medical Need ??' },
                      { msg: 'Trapped - Urgent ??' }
                    ].map((item, idx) => (
                      <button 
                        key={idx}
                        onClick={() => {
                          setBroadcasts([{
                            id: Date.now(),
                            sender: 'YOU',
                            text: item.msg,
                            time: 'Just now',
                            type: 'info'
                          }, ...broadcasts]);
                          showToast(`Broadcasted: "${item.msg}"`, 'success');
                        }}
                        className="bg-zinc-950/60 hover:bg-zinc-800 p-2.5 rounded-xl border border-zinc-800 text-xs text-left font-medium text-zinc-200 transition">
                        {item.msg}
                      </button>
                    ))}
                  </div>
                </div>

                {/* Mesh Broadcast Inbox */}
                <div className="space-y-2">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Mesh Broadcast Inbox</h3>
                  <div className="space-y-2 max-h-40 overflow-y-auto pr-1">
                    {broadcasts.map((b) => (
                      <div key={b.id} className={`p-3 rounded-xl border text-xs font-mono space-y-1 ${
                        b.type === 'alert' ? 'bg-red-950/40 border-red-800 text-red-200' : 
                        b.type === 'doctor' ? 'bg-emerald-950/40 border-emerald-800 text-emerald-200' :
                        'bg-zinc-950/60 border-zinc-800 text-zinc-300'
                      }`}>
                        <div className="flex justify-between text-[10px] text-zinc-400">
                          <span className="font-bold text-zinc-200">{b.sender}</span>
                          <span>{b.time}</span>
                        </div>
                        <p>{b.text}</p>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
            )}

            {/* TAB 2: RADAR & HAZARDS */}
            {activeTab === 'radar' && (
              <div className="space-y-5">
                <div className="bg-zinc-950/60 p-4 rounded-2xl border border-zinc-800 space-y-3">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Nearby Node Radar</h3>
                  
                  {/* Radar Screen Graphic */}
                  <div className="relative w-full h-40 rounded-xl border border-emerald-900/50 bg-black/80 overflow-hidden flex items-center justify-center">
                    <div className="absolute inset-0 border border-emerald-900/30 rounded-full scale-75"></div>
                    <div className="absolute inset-0 border border-emerald-900/20 rounded-full scale-50"></div>
                    <div className="absolute w-full h-[1px] bg-emerald-900/30"></div>
                    <div className="absolute h-full w-[1px] bg-emerald-900/30"></div>
                    
                    {/* User Node */}
                    <div className="w-3 h-3 bg-emerald-400 rounded-full animate-ping z-10"></div>
                    <div className="w-2.5 h-2.5 bg-emerald-500 rounded-full z-10 absolute"></div>

                    {/* Nodes */}
                    <div className="absolute top-6 left-10 flex items-center gap-1">
                      <span className="w-2 h-2 rounded-full bg-emerald-400"></span>
                      <span className="text-[9px] font-mono text-emerald-300">Dr. Sarah (60m)</span>
                    </div>
                    <div className="absolute bottom-8 right-8 flex items-center gap-1">
                      <span className="w-2 h-2 rounded-full bg-cyan-400"></span>
                      <span className="text-[9px] font-mono text-cyan-300">#8812 (45m)</span>
                    </div>
                  </div>

                  {/* Connected Peer List */}
                  <div className="space-y-1.5 pt-1">
                    {nodes.map((node, i) => (
                      <div key={i} className="flex justify-between items-center bg-zinc-900 p-2 rounded-lg border border-zinc-800 text-xs font-mono">
                        <div>
                          <p className="font-bold text-zinc-200">{node.id}</p>
                          <p className="text-[10px] text-zinc-500">{node.status} • {node.hops} hop(s)</p>
                        </div>
                        <div className="text-right">
                          <p className="text-emerald-400 font-bold">{node.dist}</p>
                          <p className="text-[10px] text-zinc-500">{node.signal}</p>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>

                {/* Local Hazard Reporting */}
                <div className="bg-zinc-950/60 p-4 rounded-2xl border border-zinc-800 space-y-3">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Report Obstacle / Hazard</h3>
                  <form onSubmit={(e) => {
                    e.preventDefault();
                    if(!newHazard.trim()) return;
                    setHazards([{ id: Date.now(), text: newHazard, severity: 'Warning', time: 'Just now' }, ...hazards]);
                    setNewHazard('');
                    showToast('?? Hazard Logged to Mesh Radar', 'alert');
                  }} className="flex gap-2">
                    <input 
                      type="text" 
                      placeholder="e.g. Flooded bridge at Sector 4" 
                      value={newHazard}
                      onChange={(e) => setNewHazard(e.target.value)}
                      className="flex-1 bg-zinc-900 border border-zinc-700 text-xs p-2.5 rounded-xl font-mono focus:outline-none focus:border-red-500 text-zinc-100"
                    />
                    <button type="submit" className="bg-red-700 hover:bg-red-600 text-xs px-3 rounded-xl font-bold font-mono">
                      Log
                    </button>
                  </form>

                  <div className="space-y-2 pt-1">
                    {hazards.map((h) => (
                      <div key={h.id} className="p-2.5 rounded-xl bg-amber-950/20 border border-amber-900/50 text-xs font-mono flex justify-between items-center">
                        <div>
                          <p className="text-amber-200 font-bold">?? {h.text}</p>
                          <p className="text-[10px] text-amber-500/80">{h.time}</p>
                        </div>
                        <span className="text-[10px] bg-amber-900/60 text-amber-200 px-2 py-0.5 rounded font-mono">{h.severity}</span>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
            )}

            {/* TAB 3: FIRST AID & DOCTOR DIRECTORY */}
            {activeTab === 'firstaid' && (
              <div className="space-y-4">
                {/* Emergency Doctor & Telemedicine Helplines */}
                <div className="bg-zinc-950/60 p-4 rounded-2xl border border-zinc-800 space-y-2">
                  <h3 className="text-xs font-mono uppercase text-emerald-400 font-semibold flex items-center gap-1">
                    ?? Doctor Contacts & Telemedicine
                  </h3>
                  <p className="text-[10px] text-zinc-400 font-mono">For when network connectivity is intermittently restored:</p>
                  
                  <div className="grid grid-cols-2 gap-2 pt-1 font-mono text-xs">
                    <a href="tel:112" className="bg-zinc-900 hover:bg-zinc-800 p-2.5 rounded-xl border border-zinc-800 text-center text-emerald-300 font-bold">
                      ?? Call 112 / 911
                    </a>
                    <a href="tel:108" className="bg-zinc-900 hover:bg-zinc-800 p-2.5 rounded-xl border border-zinc-800 text-center text-emerald-300 font-bold">
                      ?? Ambulance Hotline
                    </a>
                  </div>
                </div>

                {/* Acoustic Beacon Siren */}
                <div className="bg-zinc-950/60 p-3.5 rounded-2xl border border-zinc-800 space-y-2">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Acoustic Emergency Beacon</h3>
                  <div className="grid grid-cols-3 gap-1 p-1 bg-zinc-900 rounded-xl border border-zinc-800">
                    {[
                      { id: 'whistle', label: 'Whistle' },
                      { id: 'siren', label: 'Siren' },
                      { id: 'dogWhistle', label: 'Rescue Dog' }
                    ].map((tone) => (
                      <button 
                        key={tone.id}
                        onClick={() => {
                          setSirenTone(tone.id);
                          if (isSirenActive) startAudioSiren(tone.id);
                        }}
                        className={`py-1 text-[10px] font-mono rounded-lg transition ${
                          sirenTone === tone.id ? 'bg-zinc-800 text-red-400 font-bold border border-zinc-700' : 'text-zinc-500 hover:text-zinc-300'
                        }`}>
                        {tone.label}
                      </button>
                    ))}
                  </div>
                  <button 
                    onClick={toggleSiren}
                    className={`w-full py-2.5 rounded-xl font-bold text-xs tracking-wider border transition ${
                      isSirenActive ? 'bg-red-600 text-white border-red-400' : 'bg-zinc-800 hover:bg-zinc-700 text-red-400 border-red-900/60'
                    }`}>
                    {isSirenActive ? '?? STOP SIREN' : '?? ACTIVATE SOUND & STROBE'}
                  </button>
                </div>

                {/* Comprehensive Offline First Aid Guides */}
                <div className="space-y-2">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Offline First Aid Directory</h3>
                  {[
                    { title: '?? Stop Severe Bleeding', body: 'Apply direct firm pressure with clean cloth. Elevate wound above heart level. Do not remove soaked cloths; layer new ones on top.' },
                    { title: '?? CPR Protocol', body: '100-120 compressions/min in center of chest. Push 2 inches deep continuously to the beat of "Staying Alive".' },
                    { title: '?? Emergency Water Purification', body: 'Boil rapidly for 1 minute or add 2 drops of unscented household bleach per liter. Let sit for 30 mins before drinking.' },
                    { title: '?? Treating Burns', body: 'Cool burn immediately under cool water for 10-15 mins. Cover loosely with sterile non-stick bandage. Never apply ice or butter.' },
                    { title: '?? Fractures & Splinting', body: 'Immobilize injured area above and below fracture. Apply ice wrapped in cloth. Do not attempt to realign broken bones.' },
                    { title: '?? Choking (Heimlich Maneuver)', body: 'Stand behind person, lean them forward. Give 5 sharp back blows between shoulder blades, followed by 5 quick abdominal thrusts.' },
                    { title: '??? Heatstroke & Hypothermia', body: 'Heatstroke: Move to shade, cool skin with wet cloths. Hypothermia: Remove wet clothing, wrap in dry blankets, give warm fluids if conscious.' }
                  ].map((card, idx) => (
                    <div 
                      key={idx} 
                      onClick={() => setActiveFirstAid(activeFirstAid === idx ? null : idx)}
                      className="bg-zinc-950/60 p-3 rounded-xl border border-zinc-800 cursor-pointer hover:border-zinc-700 transition">
                      <div className="font-semibold text-xs flex justify-between items-center text-zinc-200">
                        <span>{card.title}</span>
                        <span className="text-zinc-500">{activeFirstAid === idx ? '?' : '?'}</span>
                      </div>
                      {activeFirstAid === idx && (
                        <p className="text-xs font-mono text-zinc-400 mt-2 border-t border-zinc-800 pt-2 leading-relaxed">{card.body}</p>
                      )}
                    </div>
                  ))}
                </div>
              </div>
            )}

            {/* TAB 4: REAL SCANNABLE QR CODE PAGE */}
            {activeTab === 'qrcode' && (
              <div className="space-y-5">
                
                {/* Real Scannable QR Badge Header */}
                <div className="bg-red-950/40 p-4 rounded-2xl border border-red-800/80 space-y-1 text-center">
                  <span className="text-[10px] font-mono bg-red-800 text-red-100 px-2.5 py-0.5 rounded uppercase font-bold">First Responder Scannable Badge</span>
                  <h2 className="text-base font-bold text-red-200 pt-1">Scan To Launch App On Phone</h2>
                  <p className="text-xs text-zinc-300 font-mono">Point any smartphone camera to open this app directly.</p>
                </div>

                {/* Dynamically Generated Real QR Code Image */}
                <div className="bg-white p-5 rounded-3xl border-4 border-red-600 shadow-[0_0_30px_rgba(220,38,38,0.3)] flex flex-col items-center justify-center space-y-3">
                  <img 
                    src={`https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${encodeURIComponent(qrTargetUrl)}`} 
                    alt="Scannable Emergency QR Code"
                    className="w-48 h-48 rounded-lg shadow-md"
                  />
                  <div className="text-center font-mono">
                    <p className="text-xs text-zinc-900 font-bold">?? Point Camera Here</p>
                    <p className="text-[10px] text-zinc-600 truncate max-w-[240px] mt-0.5">{qrTargetUrl}</p>
                  </div>
                </div>

                {/* Live Web Link Configuration */}
                <div className="bg-zinc-950/60 p-4 rounded-2xl border border-zinc-800 space-y-2">
                  <h3 className="text-xs font-mono uppercase text-zinc-400 font-semibold">Active Web Address</h3>
                  <p className="text-[10px] text-zinc-500 font-mono">When published online, this automatically locks onto your live website link:</p>
                  <input 
                    type="url" 
                    value={qrTargetUrl} 
                    onChange={(e) => setQrTargetUrl(e.target.value)}
                    className="w-full bg-zinc-900 border border-zinc-700 text-xs p-2.5 rounded-xl font-mono text-zinc-100 focus:outline-none focus:border-red-500"
                    placeholder="https://your-beacon-app.netlify.app"
                  />
                </div>

                {/* Interactive Preview of What Scanners See */}
                <div className="bg-emerald-950/30 p-4 rounded-2xl border border-emerald-800/60 space-y-3">
                  <h3 className="text-xs font-mono uppercase text-emerald-300 font-bold flex items-center gap-1">
                    ?? What Scanners Will See: Calm & Emergency Guide
                  </h3>

                  {/* Interactive Box Breathing Timer */}
                  <div className="bg-zinc-950 p-4 rounded-xl border border-emerald-900/50 text-center space-y-2">
                    <p className="text-[10px] font-mono text-emerald-400 uppercase tracking-widest">Guided Box Breathing</p>
                    <div className="text-lg font-black text-emerald-200 font-mono animate-pulse">{breathingPhase}</div>
                    <p className="text-[11px] text-zinc-400 font-mono">4s In • 4s Hold • 4s Out • 4s Hold</p>
                  </div>

                  {/* 5-4-3-2-1 Grounding Method */}
                  <div className="space-y-1.5 font-mono text-xs text-zinc-300">
                    <p className="font-bold text-emerald-300">5-4-3-2-1 Grounding Technique:</p>
                    <ul className="list-disc list-inside text-[11px] space-y-1 text-zinc-400">
                      <li><strong className="text-zinc-200">5 things</strong> you can see around you</li>
                      <li><strong className="text-zinc-200">4 things</strong> you can physically feel</li>
                      <li><strong className="text-zinc-200">3 things</strong> you hear</li>
                      <li><strong className="text-zinc-200">2 things</strong> you can smell</li>
                      <li><strong className="text-zinc-200">1 deep breath</strong> into your diaphragm</li>
                    </ul>
                  </div>

                  {/* Direct Emergency Services Hotlines */}
                  <div className="pt-2">
                    <p className="text-xs font-mono font-bold text-zinc-300 mb-2">Direct One-Tap Hotline Buttons:</p>
                    <div className="grid grid-cols-2 gap-2 text-xs font-mono">
                      <a href="tel:112" className="bg-red-950/80 hover:bg-red-900 border border-red-700 p-2.5 rounded-xl text-center text-red-200 font-bold">
                        ?? 112 / 911 <span className="block text-[9px] text-red-300 font-normal">Universal Call</span>
                      </a>
                      <a href="tel:108" className="bg-zinc-900 hover:bg-zinc-800 border border-zinc-700 p-2.5 rounded-xl text-center text-zinc-200 font-bold">
                        ?? Ambulance <span className="block text-[9px] text-zinc-400 font-normal">Medical Triage</span>
                      </a>
                    </div>
                  </div>
                </div>

              </div>
            )}

          </main>

          {/* Bottom Navigation Tabs */}
          <nav className="border-t border-zinc-800 bg-zinc-950/90 p-2 grid grid-cols-4 gap-1">
            <button 
              onClick={() => setActiveTab('comms')}
              className={`py-2 text-[10px] font-bold rounded-xl transition font-mono ${
                activeTab === 'comms' ? 'bg-zinc-800 text-red-400 border border-zinc-700' : 'text-zinc-500 hover:text-zinc-300'
              }`}>
              ?? Comms
            </button>
            <button 
              onClick={() => setActiveTab('radar')}
              className={`py-2 text-[10px] font-bold rounded-xl transition font-mono ${
                activeTab === 'radar' ? 'bg-zinc-800 text-red-400 border border-zinc-700' : 'text-zinc-500 hover:text-zinc-300'
              }`}>
              ??? Radar
            </button>
            <button 
              onClick={() => setActiveTab('firstaid')}
              className={`py-2 text-[10px] font-bold rounded-xl transition font-mono ${
                activeTab === 'firstaid' ? 'bg-zinc-800 text-red-400 border border-zinc-700' : 'text-zinc-500 hover:text-zinc-300'
              }`}>
              ?? First Aid
            </button>
            <button 
              onClick={() => setActiveTab('qrcode')}
              className={`py-2 text-[10px] font-bold rounded-xl transition font-mono ${
                activeTab === 'qrcode' ? 'bg-red-950 text-red-300 border border-red-700' : 'text-zinc-500 hover:text-zinc-300'
              }`}>
              ?? Scan QR
            </button>
          </nav>

        </div>
      );
    }

    ReactDOM.render(<BeaconApp />, document.getElementById('root'));
  </script>
</body>
</html>
