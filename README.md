<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>أكاديمية الفرنسية الذكية - نظام الاختبارات</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
    <style>
        :root { --main-blue: #002395; --light-bg: #f8f9fa; }
        body { background-color: var(--light-bg); font-family: sans-serif; }
        .roadmap-container { position: relative; padding: 40px 0; }
        .roadmap-line { position: absolute; left: 50%; top: 0; bottom: 0; width: 4px; background: #ddd; transform: translateX(-50%); z-index: 1; }
        .level-node { position: relative; z-index: 2; margin-bottom: 50px; display: flex; flex-direction: column; align-items: center; }
        .level-circle { width: 70px; height: 70px; border-radius: 50%; background: white; border: 4px solid #ddd; display: flex; align-items: center; justify-content: center; font-size: 24px; color: #999; cursor: pointer; transition: 0.3s; }
        .active .level-circle { border-color: var(--main-blue); color: var(--main-blue); box-shadow: 0 0 15px rgba(0,35,149,0.2); }
        .level-info { background: white; border-radius: 12px; padding: 8px 15px; margin-top: 10px; box-shadow: 0 4px 10px rgba(0,0,0,0.05); text-align: center; }
        .ai-card { background: white; border-radius: 20px; border: 2px solid var(--main-blue); padding: 25px; margin-top: 30px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); }
        .quiz-option { cursor: pointer; transition: 0.2s; border: 2px solid #eee; border-radius: 10px; padding: 12px; margin-bottom: 10px; text-align: right; }
        .quiz-option:hover { background-color: #f0f7ff; border-color: var(--main-blue); }
    </style>
</head>
<body>

<nav class="navbar navbar-light bg-white shadow-sm sticky-top">
    <div class="container">
        <span class="navbar-brand fw-bold text-primary"><i class="fas fa-brain"></i> Français AI Quiz</span>
        <div class="badge bg-warning text-dark p-2">⭐ النقاط: <span id="userPoints">0</span></div>
    </div>
</nav>

<div class="container mt-4">
    <div class="roadmap-container" id="levelsMap"></div>

    <div class="ai-card mb-5">
        <h5 class="text-center mb-3">مصحح الجمل (احصد 5 نقاط)</h5>
        <textarea id="aiInput" class="form-control mb-3" rows="2" placeholder="اكتب جملة فرنسية لتصحيحها..."></textarea>
        <button onclick="checkSentence()" id="btnCheck" class="btn btn-primary w-100 fw-bold">تحقق الآن</button>
        <div id="aiResponse" class="mt-3 p-3 bg-light rounded d-none" style="white-space: pre-wrap;"></div>
    </div>
</div>

<div class="modal fade" id="quizModal" tabindex="-1" aria-hidden="true">
    <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content" style="border-radius: 20px;">
            <div class="modal-header border-0">
                <h5 class="modal-title" id="quizTitle">اختبار المستوى</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <div id="quizLoading" class="text-center">
                    <div class="spinner-border text-primary mb-2"></div>
                    <p>جاري توليد أسئلة ذكية لك...</p>
                </div>
                <div id="quizContent" class="d-none">
                    <p id="questionText" class="fw-bold fs-5 mb-4"></p>
                    <div id="optionsList"></div>
                </div>
            </div>
        </div>
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
<script>
    const API_KEY = "AIzaSyC8xFjYovaYTFY5CT2pX3xI_Zd-QSRyu_c";
    let points = parseInt(localStorage.getItem('fr_quiz_pts')) || 0;

    const levels = [
        { id: 1, title: "الأساسيات A1", req: 0, topic: "Greetings and Numbers" },
        { id: 2, title: "الحياة اليومية A2", req: 50, topic: "Restaurant and Shopping" },
        { id: 3, title: "المستوى المتوسط B1", req: 150, topic: "Work and Future Tense" },
        { id: 4, title: "المستوى المتقدم B2", req: 300, topic: "Politics and Environment" }
    ];

    function updateUI() {
        document.getElementById('userPoints').innerText = points;
        const map = document.getElementById('levelsMap');
        map.innerHTML = '<div class="roadmap-line"></div>';

        levels.forEach(level => {
            const isLocked = points < level.req;
            map.innerHTML += `
                <div class="level-node ${isLocked ? '' : 'active'}">
                    <div class="level-circle" onclick="startQuiz(${level.id}, ${isLocked})">
                        ${isLocked ? '<i class="fas fa-lock"></i>' : level.id}
                    </div>
                    <div class="level-info">
                        <div class="fw-bold small">${level.title}</div>
                        ${isLocked ? `<div class="text-danger x-small">مغلق (${level.req}⭐)</div>` : '<div class="text-success x-small">ابدأ الاختبار</div>'}
                    </div>
                </div>`;
        });
    }

    // وظيفة التصحيح العادية لجني النقاط
    async function checkSentence() {
        const input = document.getElementById('aiInput').value;
        if (!input) return;
        const btn = document.getElementById('btnCheck');
        btn.disabled = true; btn.innerText = 'جاري التحليل...';

        try {
            const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${API_KEY}`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ contents: [{ parts: [{ text: `Correct this French sentence and explain in Arabic: "${input}"` }] }] })
            });
            const data = await res.json();
            document.getElementById('aiResponse').innerText = data.candidates[0].content.parts[0].text;
            document.getElementById('aiResponse').classList.remove('d-none');
            
            points += 5;
            savePoints();
            Swal.fire('جميل!', 'حصلت على 5 نقاط!', 'success');
        } catch (e) { Swal.fire('خطأ', 'تأكد من الإنترنت ومسار الملف في Chrome', 'error'); }
        btn.disabled = false; btn.innerText = 'تحقق الآن';
    }

    // نظام الاختبار الذكي
    let currentLevelTopic = "";
    async function startQuiz(levelId, isLocked) {
        if (isLocked) return Swal.fire('مغلق', 'اجمع المزيد من النقاط لفتح هذا المستوى!', 'warning');
        
        const level = levels.find(l => l.id === levelId);
        currentLevelTopic = level.topic;
        
        const modal = new bootstrap.Modal(document.getElementById('quizModal'));
        modal.show();
        
        document.getElementById('quizLoading').classList.remove('d-none');
        document.getElementById('quizContent').classList.add('d-none');

        try {
            const prompt = `Generate ONE multiple choice question in French for a student at level ${level.title} about ${level.topic}. 
            Format the response strictly as JSON like this: 
            {"question": "the question", "options": ["opt1", "opt2", "opt3", "opt4"], "correct": 0, "explanation": "explanation in Arabic"}`;

            const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${API_KEY}`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] })
            });
            const data = await res.json();
            const quizData = JSON.parse(data.candidates[0].content.parts[0].text.replace(/```json|```/g, ''));
            
            showQuestion(quizData);
        } catch (e) {
            modal.hide();
            Swal.fire('عذراً', 'فشل في توليد السؤال، حاول مرة أخرى.', 'error');
        }
    }

    function showQuestion(data) {
        document.getElementById('quizLoading').classList.add('d-none');
        document.getElementById('quizContent').classList.remove('d-none');
        
        document.getElementById('questionText').innerText = data.question;
        const list = document.getElementById('optionsList');
        list.innerHTML = '';
        
        data.options.forEach((opt, index) => {
            const div = document.createElement('div');
            div.className = 'quiz-option';
            div.innerText = opt;
            div.onclick = () => checkAnswer(index, data.correct, data.explanation);
            list.appendChild(div);
        });
    }

    function checkAnswer(selected, correct, explanation) {
        const modal = bootstrap.Modal.getInstance(document.getElementById('quizModal'));
        modal.hide();

        if (selected === correct) {
            points += 20;
            savePoints();
            Swal.fire({
                title: 'إجابة صحيحة! (+20⭐)',
                text: explanation,
                icon: 'success'
            });
        } else {
            Swal.fire({
                title: 'للأسف، إجابة خاطئة',
                text: explanation,
                icon: 'error'
            });
        }
    }

    function savePoints() {
        localStorage.setItem('fr_quiz_pts', points);
        updateUI();
    }

    updateUI();
</script>

</body>
</html>
