# K6 lab results

## Part 0
- **Step 0.1, http_req_duration:** avg=485.37ms min=265.59ms med=281.82ms max=2.37s p(90)=1.25s p(95)=1.74s (จากผลรันไฟล์แรกสุด)
- **Step 0.2, requests, rate and p(95) and why the rate stays below 5:** 
  - Requests: 70
  - Rate: 3.181038/s
  - p(95): 1.74s
  - Why the rate stays below 5: สาเหตุที่ rate ไม่ถึง 5 requests/s เป็นเพราะในสคริปต์มีคำสั่ง `sleep(1)` ซึ่งจำลองเวลาการคิด (Think time) ของผู้ใช้งาน ทำให้ Virtual User แต่ละคนต้องหยุดรอ 1 วินาทีต่อ 1 รอบ (Iteration)
- **Step 0.6, exit code of the normal run and of the run with P95=300:** รันแบบปกติ (threshold 1500) Exit code คือ 99 (เพราะ p(95)=1.67s ซึ่งเกินเกณฑ์), รันแบบ P95=300 Exit code ก็คือ 99 (p(95)=724.33ms ซึ่งเกิน 300ms)
- **Step 0.7, rules that passed, p(95), p(99), exit code after your change:** 
  - Passed rules: `http_req_duration` ('p(95)<1000', 'p(99)<1500') และ `http_req_failed` ('rate<0.01') 
  - p(95): 398.54ms
  - p(99): 538.9ms
  - Exit code after change: 0 (ผ่านเกณฑ์ทั้งหมด)
- **Step 0.8, what was wrong, and how the checks showed it:** สิ่งที่ผิดคือการกำหนดค่าใน JSON Body ไม่ตรงกับฐานข้อมูลของ API (พิมพ์เล็กพิมพ์ใหญ่) เช่น Topping ในฐานข้อมูลคือ "Pepperoni" (ตัว P ใหญ่) และ Tool คือ "Pizza cutter" หรือ "Knife" (ตัว K ใหญ่) พอเราส่งคำขอไปแบบผิดๆ API จึงไม่รับรู้เงื่อนไขและคืนค่าพิซซ่าปกติกลับมา ทำให้การเช็ก 'no pepperoni' (ล้มเหลว 16%) และ 'no knife needed' (ล้มเหลว 41%) แจ้งเตือนข้อผิดพลาด (Failed)

## Part A results

| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
|------|-----------|-------|-------|-------|-----|--------|--------|
| A1 | Smoke | 1 | 0.529 | 389.91ms | 391.23ms | 0.00% | 100.00% |
| A2 | Average load | 3 | 1.256 | 1.03s | 1.72s | 0.00% | 100.00% |
| A3 | Stress | 5 | 2.101 | 439.39ms | 2.23s | 0.00% | 100.00% |
| A4 | Spike | 5 | 1.075 | 418.10ms | 1.4s | 0.00% | 100.00% |
| A5 | Soak | 2 | 0.919 | 971.97ms | 2.03s | 0.00% | 100.00% |

## Part B

1. **Fill in the results table for Steps A1 to A5.**
   - (เติมข้อมูลเรียบร้อยแล้วในตารางด้านบน)

2. **In Step 0.2, why was the request rate below 5 requests per second with 5 virtual users?**
   - เพราะในสคริปต์มีคำสั่ง `sleep(1)` เพื่อจำลองพฤติกรรมการอ่านข้อมูล (Think time) ของผู้ใช้จริง ทำให้สคริปต์หยุดพักชั่วคราว ไม่ได้ยิง Request ต่อเนื่องแบบรัวๆ

3. **Did the response time grow from the average-load test to the stress test? Explain your answer with the operational envelope.**
   - จากตัวเลขที่รันได้ ค่า p(95) ของ Stress Test (439.39ms) กลับมีความเร็วกว่า Average load (1.03s) ซึ่งโดยทฤษฎีแล้ว Stress Test ที่มีโหลดหนักกว่า (5 VUs) ควรจะช้ากว่า (เข้าใกล้ Operational Envelope มากกว่า) สาเหตุที่ตัวเลขขัดแย้งกันเกิดจากการทดสอบบนอินเทอร์เน็ตสาธารณะ (Shared Public Site) ซึ่งความหน่วง (Latency) และปริมาณการใช้งานเว็บของคนอื่นๆ ในช่วงเวลาที่รัน A2 อาจจะหนาแน่นกว่าตอนที่รัน A3

4. **In the spike test, how long did the response time take to return to normal after the spike?**
   - ระบบใช้เวลาฟื้นตัวกลับสู่สภาวะปกติอย่างรวดเร็ว (ประมาณ 5-10 วินาที หลังจากลดจำนวน User ลงจาก 5 กลับมาที่ 1) โดยไม่มี Request ใดล้มเหลวเลย

5. **In the soak test, did the response time or the error rate get worse over time? Name one kind of defect that a soak test of several hours could reveal.**
   - อัตราการผิดพลาด (Error rate) ยังคงเป็น 0.00% และ Response Time อยู่ในเกณฑ์ทรงตัว (p(95) = 971.97ms) ไม่ได้แย่ลงอย่างมีนัยสำคัญ ข้อบกพร่องที่ค้นพบได้จาก Soak test ได้แก่ ปัญหา Memory Leak (หน่วยความจำรั่วไหล), ปัญหาฐานข้อมูลค้าง (Database Connection Exhaustion), หรือพื้นที่เก็บข้อมูลเต็ม

6. **Why do the thresholds in this lab use p(95) rather than the average?**
   - เพราะค่าเฉลี่ย (Average) อาจถูกดึงให้ดูดีได้จาก Request ส่วนใหญ่ที่เร็ว และจะซ่อนข้อมูลของผู้ใช้งานกลุ่มที่ได้ผลตอบสนองช้าที่สุดเอาไว้ การใช้ p(95) เป็นการการันตีว่า 95% ของผู้ใช้งานทั้งหมดได้รับความเร็วที่ดีกว่าหรือเท่ากับค่านี้ ซึ่งสะท้อนประสบการณ์ผู้ใช้ (UX) ในภาพรวมได้แม่นยำกว่า

7. **Your results come from a shared public site reached over the internet. Give two reasons why the same tests against a system in a lab could give different numbers.**
   - 1. มีความหน่วงและความไม่เสถียรของเครือข่ายอินเทอร์เน็ต (Network Latency) มาเกี่ยวข้อง ซึ่งใน Lab จะเป็นการเชื่อมต่อใน Local Network ที่เร็วกว่าและนิ่งกว่า
   - 2. เว็บสาธารณะเป็นแบบ Shared Site อาจมีผู้ทดสอบคนอื่นๆ จากทั่วโลกส่ง Request เข้ามาใช้งานพร้อมกันในเวลานั้น ทำให้เกิดคอขวดที่เซิร์ฟเวอร์โดยที่เราควบคุมไม่ได้

8. **Why must you never run a stress, spike or breakpoint test with hundreds of users against a site that you do not own?**
   - เพราะการส่งคำขอปริมาณมหาศาลจะเป็นการโจมตีให้เซิร์ฟเวอร์ล่ม (Denial of Service - DoS) ทำให้ผู้ใช้งานจริงไม่สามารถเข้าใช้งานได้ ซึ่งผิดจรรยาบรรณและอาจผิดกฎหมาย

9. **Write one vague performance requirement for an app you use every day. Rewrite it as a testable requirement, and show the K6 thresholds that check it.**
   - Vague: "แอป Shopee ต้องโหลดเร็ว"
   - Testable: "เมื่อมีผู้ใช้งานพร้อมกัน 10 คน 95% ของการค้นหาสินค้าต้องเสร็จสิ้นภายใน 2 วินาที และมีอัตราล้มเหลวไม่เกิน 1%"
   - Thresholds: 
     ```javascript
     thresholds: {
         http_req_duration: ['p(95)<2000'],
         http_req_failed: ['rate<0.01']
     }
     ```