# Learning Log

บันทึกสิ่งที่เรียนรู้ระหว่างทำโปรเจกต์ (สร้างโดย SWE Mentor Mode)

---

## [2026-09-15] Conference Event Planner — Task 5: Meals selection (state)

**Stack:** JavaScript / React / Redux Toolkit

**Concept:** `createSlice` + Immer — "เขียนเหมือน mutate แต่ได้ immutable update"

**อธิบาย:**
- `createSlice` ของ Redux Toolkit รับ `initialState` เป็นอะไรก็ได้ — number, object, หรือ **array** (เคสนี้ใช้ array ของ meal object) และคืน `reducer` + `actions` ที่ generate ให้อัตโนมัติ จึงไม่ต้องเขียน action type string / switch-case เองแบบ Redux ยุคเก่า
- โค้ดใน reducer อย่าง `state[action.payload].selected = !state[action.payload].selected` **ดูเหมือน** mutate state ตรง ๆ ซึ่งผิดกฎ Redux ("reducers must not mutate state") แต่ RTK ห่อ reducer ด้วย **Immer** ไว้แล้ว — สิ่งที่ reducer ได้รับจริงคือ *draft proxy* ไม่ใช่ state ตัวจริง Immer จะบันทึกการแก้ไขแล้วสร้าง state object ใหม่ให้เอง ผลลัพธ์จึงยังเป็น immutable update ที่ React/Redux ตรวจจับการเปลี่ยนแปลงได้ถูกต้อง
- ข้อควรระวัง: ในไฟล์ที่ **ไม่ได้** อยู่ใน RTK reducer (เช่นเขียน reducer มือเปล่า หรือแก้ state ใน component) กฎเดิมยังบังคับอยู่ — ต้อง copy ก่อนแก้เสมอ
- `selected: false` ถูกเก็บไว้ใน slice (ไม่ใช่ `useState` ใน component) เพราะข้อมูลนี้ต้องถูกอ่านข้าม component — หน้าเลือกเมนู, แถบราคารวม และหน้าสรุป (Show Details) ใช้ชุดข้อมูลเดียวกัน

**ทางเลือกที่พิจารณา / Trade-off:**
- **Array + index เป็น payload** (ที่ใช้อยู่): เขียนสั้น ใช้ `.map((item, index) => ...)` ตรงไปตรงมา แต่ผูกกับลำดับใน array — ถ้าอนาคตมีการ sort/filter/ลบรายการ index จะเพี้ยน
- **Object keyed by id** (เช่น `{ breakfast: {...} }`) หรือ array + payload เป็น `id`: ทนต่อการเปลี่ยนลำดับมากกว่า เป็นแนวที่ Redux แนะนำสำหรับข้อมูลชุดใหญ่ (normalized state) แต่ในแล็บที่รายการคงที่ 4 ตัว ถือว่า over-engineering
- **เก็บ `selected` ใน component ด้วย `useState`**: ง่ายกว่าแต่ share ข้ามหน้าไม่ได้ ต้อง lift state up อยู่ดี

**อ้างอิง:**
- [Redux Toolkit — createSlice](https://redux-toolkit.js.org/api/createSlice)
- [Redux Toolkit — Writing Reducers with Immer](https://redux-toolkit.js.org/usage/immer-reducers)
- [Redux Essentials — Normalizing State Shape](https://redux.js.org/usage/structuring-reducers/normalizing-state-shape)

**ลองต่อยอด:** ลองเขียน `toggleMealSelection` เวอร์ชัน "ไม่พึ่ง Immer" ด้วย `state.map(...)` แล้วเทียบดูว่าโค้ดยาวขึ้นแค่ไหน — จะเห็นชัดว่า Immer ช่วยอะไร

---

## [2026-09-15] Conference Event Planner — Task 5.2: toggleMealSelection reducer

**Stack:** JavaScript / React / Redux Toolkit

**Concept:** Auto-generated action creators + รูปร่างของ action (Flux Standard Action)

**อธิบาย:**
- ทุก key ใน `reducers: {}` ของ `createSlice` จะถูก generate เป็น **action creator** ชื่อเดียวกันให้อัตโนมัติ ผ่าน `mealsSlice.actions` — จึง `export const { toggleMealSelection } = mealsSlice.actions` ได้เลย ไม่ต้องเขียน action type string เอง
- เวลาเรียก `dispatch(toggleMealSelection(2))` สิ่งที่วิ่งเข้า store คือ object `{ type: "meals/toggleMealSelection", payload: 2 }` — argument ตัวแรกที่ส่งเข้า action creator จะกลายเป็น `payload` ตรง ๆ (ถ้าอยากแปลงร่างก่อน ต้องใช้ `prepare` callback) ส่วน type string ใช้รูปแบบ `<sliceName>/<reducerName>` ซึ่งทำให้ debug ใน Redux DevTools อ่านง่าย
- รูปแบบ `{ type, payload }` นี้คือ **Flux Standard Action (FSA)** ซึ่งเป็น convention ที่ RTK ยึดตาม ไม่ใช่กฎบังคับของ Redux เอง
- Pattern `x = !x` (toggle) ดีกว่าการส่งค่า `true/false` เข้ามาเอง เพราะ **ไม่ต้องให้ UI รู้ state ปัจจุบัน** — UI แค่บอกว่า "แถวนี้ถูกคลิก" แล้วปล่อยให้ reducer เป็นเจ้าของ logic การเปลี่ยนสถานะ (single source of truth)

**ทางเลือกที่พิจารณา / Trade-off:**
- `toggleMealSelection(index)` (toggle ฝั่ง reducer): UI โง่ ๆ ได้ ไม่มีโอกาส race กับค่าเก่า
- `setMealSelection({ index, value })` (ส่งค่าใหม่มาเลย): ยืดหยุ่นกว่าถ้าวันหลังต้อง "เลือกทั้งหมด / ล้างทั้งหมด" แต่ผู้เรียกต้องอ่าน state ปัจจุบันก่อนเสมอ เสี่ยงใช้ค่าที่ค้าง

**อ้างอิง:**
- [Redux Toolkit — createAction](https://redux-toolkit.js.org/api/createAction) (รูปร่าง `{ type, payload }` และ prepare callback)
- [Redux Toolkit — createSlice](https://redux-toolkit.js.org/api/createSlice) (การ generate actions จาก key ของ reducers)
- [Flux Standard Action spec](https://github.com/redux-utilities/flux-standard-action)

**ลองต่อยอด:** เปิด Redux DevTools ตอนกด checkbox แล้วดู action ที่วิ่งเข้ามา — จะเห็น `meals/toggleMealSelection` พร้อม payload และ state diff ก่อน/หลัง

---

## [2026-09-15] Conference Event Planner — Task 5.4: ต่อ reducer เข้า store

**Stack:** JavaScript / React / Redux Toolkit

**Concept:** `configureStore` — key ใน `reducer` map คือรูปร่างของ global state

**อธิบาย:**
- object ที่ส่งเข้า `reducer: { venue, av, meals }` ถูกส่งต่อให้ `combineReducers` อัตโนมัติ **key ที่ตั้งชื่อไว้ = path ของ state** ดังนั้น `meals: mealsReducer` ทำให้อ่านค่าได้ด้วย `useSelector((state) => state.meals)` ถ้าเปลี่ยน key เป็น `mealItems` selector ก็ต้องเปลี่ยนตาม — ชื่อ key ไม่จำเป็นต้องตรงกับ `name: "meals"` ใน `createSlice` (ตัวนั้นใช้ตั้ง prefix ของ action type เท่านั้น) แต่ **ควรตั้งให้ตรงกัน** เพื่อไม่ให้สับสนตอน debug
- แต่ละ slice reducer เห็นเฉพาะ state ก้อนของตัวเอง — `mealsSlice` ได้รับ array ของ meals ไม่ใช่ state ทั้งก้อน จึงเขียน `state[action.payload]` ได้ตรง ๆ
- ทุก action ที่ dispatch จะถูกส่งให้ **ทุก** reducer เสมอ ไม่ใช่แค่ slice ที่เป็นเจ้าของ — ตัวที่ไม่รู้จัก action นั้นก็คืน state เดิม (นี่คือเหตุผลที่ action type ต้องไม่ชนกัน จึง prefix ด้วยชื่อ slice)
- `configureStore` ยัง set ให้ฟรีอีก 3 อย่างที่ Redux ดั้งเดิมต้องต่อเอง: **thunk middleware**, **dev-only middleware ที่จับ accidental mutation / non-serializable value**, และ **Redux DevTools** (ปิดอัตโนมัติใน production)

**ทางเลือกที่พิจารณา / Trade-off:**
- **Redux ดั้งเดิม** (`createStore` + `combineReducers` + `applyMiddleware` + `composeWithDevTools` เอง): คุมได้ละเอียดแต่ boilerplate เยอะและพลาดง่าย
- **แยกหลาย store**: Redux ออกแบบมาให้มี store เดียวต่อแอป — การแยก store ทำให้ share state ข้ามกันไม่ได้

**อ้างอิง:**
- [Redux Toolkit — configureStore](https://redux-toolkit.js.org/api/configureStore)
- [Redux Toolkit — getDefaultMiddleware](https://redux-toolkit.js.org/api/getDefaultMiddleware)
- [Redux — combineReducers](https://redux.js.org/api/combinereducers)

**ลองต่อยอด:** ลองพิมพ์ `store.getState()` ใน console ดูรูปร่าง state ทั้งก้อน — จะเห็น `{ venue: [...], av: [...], meals: [...] }` ตรงกับ key ที่ตั้งไว้เป๊ะ

---

## [2026-09-15] Conference Event Planner — Task 5.6: useSelector

**Stack:** JavaScript / React / React-Redux

**Concept:** `useSelector` = subscribe + reference equality (`===`)

**อธิบาย:**
- `useSelector` ไม่ใช่แค่การอ่านค่า แต่เป็นการ **subscribe** component เข้ากับ store — ทุกครั้งที่มี action ถูก dispatch (action ไหนก็ได้ ไม่ใช่แค่ของ slice นั้น) React-Redux จะรัน selector ใหม่แล้วเทียบผลกับรอบก่อน
- การเทียบใช้ **strict reference equality (`===`)** ไม่ใช่ deep compare → selector ที่คืน object/array **ใหม่** ทุกครั้ง จะทำให้ re-render ทุก action
  ```js
  useSelector((s) => s.meals)                        // ✅ reference เดิม
  useSelector((s) => s.meals.filter((m) => m.selected)) // ❌ array ใหม่ทุกครั้ง
  ```
- ทางแก้: select ก้อนดิบมาแล้วค่อย derive นอก selector, เรียก `useSelector` หลายครั้งแยกค่า, หรือใช้ memoized selector (Reselect / `createSelector`)

**ทางเลือกที่พิจารณา / Trade-off:** select ก้อนใหญ่ = re-render บ่อยกว่าที่จำเป็นเมื่อ field ที่ไม่สนใจเปลี่ยน แต่โค้ดง่ายกว่า; select แคบหลายตัว = re-render น้อยลงแต่โค้ดยาวขึ้น — ในแอปเล็กแบบนี้เลือกอย่างแรก

**อ้างอิง:**
- [React-Redux — useSelector](https://react-redux.js.org/api/hooks#useselector)
- [React-Redux — memoizing selectors](https://react-redux.js.org/api/hooks#using-memoizing-selectors)

**ลองต่อยอด:** ลองใส่ `console.log("render")` ใน component แล้วกดปุ่ม venue ดู — component จะ re-render แม้ meals ไม่เปลี่ยน เพราะ `venueItems` เปลี่ยน reference

---

## [2026-09-15] Conference Event Planner — Task 5.7: Controlled input

**Stack:** JavaScript / React

**Concept:** Controlled component — state เป็น single source of truth ของค่าใน `<input>`

**อธิบาย:**
- `<input value={numberOfPeople} onChange={...} />` คือ **controlled input** — React บังคับให้ค่าใน DOM ตรงกับ state เสมอ ถ้าไม่เรียก `setState` ใน `onChange` ค่าที่พิมพ์จะถูก revert ทันทีทุก keystroke (React จะเตือน "You provided a `value` prop to a form field without an `onChange` handler")
- ข้อดีคือ **validate/normalize ได้ก่อนค่าจะเข้า state** — โค้ดนี้ทำ `parseInt` แล้ว clamp ค่าต่ำกว่า 1 หรือ `NaN` ให้เป็น 1 → ค่าใน state จึงเป็นตัวเลข ≥ 1 เสมอ ไม่ต้องไปกันพลาดตอนคำนวณราคา
- `min="1"` เป็นการกันฝั่ง **HTML validation** อย่างเดียว (กันปุ่ม spinner ลงต่ำกว่า 1) แต่ผู้ใช้ยังพิมพ์ `-5` ลงไปตรง ๆ ได้ → **ต้องมี guard ใน JS ด้วยเสมอ** ห้ามเชื่อ attribute ของ HTML อย่างเดียว (หลักการเดียวกับ never trust client-side validation ฝั่ง server)
- `htmlFor` คือ `for` ของ HTML (คำว่า `for` เป็น reserved word ใน JS) — การผูก label เข้ากับ input ผ่าน `htmlFor`/`id` ทำให้คลิกที่ label แล้ว focus เข้า input และ screen reader อ่านชื่อ field ได้ถูกต้อง
- ห้ามสลับ controlled ↔ uncontrolled กลางคัน (`value={undefined}` แล้วค่อยเป็น string) React จะ error — ค่า initial ต้องไม่เป็น `null`/`undefined`

**ทางเลือกที่พิจารณา / Trade-off:**
- **Controlled** (ที่ใช้อยู่): validate ได้ทันที, state เป็นแหล่งความจริงเดียว แต่ re-render ทุก keystroke
- **Uncontrolled** (`defaultValue` + `useRef`): re-render น้อยกว่า เหมาะกับฟอร์มใหญ่มาก ๆ หรือ integrate กับ non-React code แต่ validate/sync ยากกว่า
- **เก็บ `numberOfPeople` ใน Redux แทน `useState`**: จำเป็นก็ต่อเมื่อ component อื่นต้องอ่านค่านี้ — ในแล็บนี้ใช้เฉพาะในไฟล์เดียว จึงเก็บเป็น local state ถูกต้องแล้ว (หลักการ: อย่ายก state ขึ้น global ถ้าไม่มีใคร share)

**ข้อสังเกต (UX quirk):** guard ตัวนี้ทำให้ **ลบค่าในช่องจนว่างไม่ได้** — พอกด backspace ตัวสุดท้าย ค่าจะเด้งเป็น 1 ทันที ถ้าอยากให้ลบจนว่างได้ ต้องยอมให้ state เก็บ `""` ชั่วคราวแล้วค่อย normalize ตอน blur/submit

**อ้างอิง:**
- [React — `<input>` (controlled inputs)](https://react.dev/reference/react-dom/components/input)
- [MDN — `<input type="number">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/number)
- [MDN — `<label>` และการผูกด้วย `for`/`id`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/label)

**ลองต่อยอด:** ลองเอา `onChange` ออกแล้วพิมพ์ดู — จะพิมพ์ไม่ได้เลย และ console จะขึ้น warning ของ React เรื่อง read-only field

---

## [2026-09-15] Conference Event Planner — Task 5.8: render list ด้วย map() + controlled checkbox

**Stack:** JavaScript / React

**Concept:** `key` ใน list, controlled checkbox (`checked`), และ event handler ที่ต้องส่ง argument

**อธิบาย:**
- **Controlled checkbox ใช้ `checked` ไม่ใช่ `value`** — คู่กับ `onChange` เหมือน text input ทุกอย่าง: `checked={item.selected}` ทำให้ DOM สะท้อน Redux state เสมอ ถ้า handler ไม่เปลี่ยน state ติ๊กแล้วจะเด้งกลับทันที
- **`onChange={() => handleMealSelection(index)}` ต้องห่อด้วย arrow function** — ถ้าเขียน `onChange={handleMealSelection(index)}` จะเป็นการ **เรียกฟังก์ชันทันทีตอน render** แล้วเอา return value (undefined) ไปเป็น handler → dispatch รัวตอนเรนเดอร์จนเกิด infinite loop
- **`key` ต้อง unique ในหมู่พี่น้องและต้องนิ่ง** React ใช้ key จับคู่ element ข้ามรอบ render ถ้า key เปลี่ยน React จะทิ้งของเก่าสร้างใหม่หมด (ช้า + state ใน element หาย) — `key={index}` ใช้ได้เฉพาะลิสต์ที่ **ไม่มีการ insert/delete/reorder** อย่างเมนู 4 อย่างนี้ ถ้าลิสต์ dynamic ต้องใช้ id จริงจากข้อมูล
- **`id={\`meal_${index}\`}` ต้องไม่ซ้ำ** เพราะ `<label htmlFor>` จับคู่ด้วย `id` — ถ้า hardcode เป็น `id="meal"` ทั้ง 4 อัน คลิก label ไหนก็จะไปติ๊ก checkbox ตัวแรกเสมอ (bug ที่เจอบ่อยมากเวลาทำ list ของ form field)
- **`style={{ padding: 15 }}` วงเล็บปีกกาสองชั้น** = ชั้นนอกคือ JSX expression ชั้นในคือ object literal; React เติม `px` ให้เองเมื่อค่าเป็น number ยกเว้น unitless property (`opacity`, `zIndex`, `lineHeight`)

**ทางเลือกที่พิจารณา / Trade-off:**
- `key={index}` (ที่ใช้): ง่าย พอสำหรับลิสต์คงที่ | `key={item.name}` : ทนต่อการ reorder กว่า และในเคสนี้ชื่อเมนูก็ unique อยู่แล้ว — เป็นตัวเลือกที่ดีกว่าถ้าจะทำจริงจัง
- Controlled checkbox: state เป็นแหล่งความจริงเดียว, sync กับ Redux ได้ | Uncontrolled (`defaultChecked`): DOM เก็บค่าเอง อ่านยากและ sync กับ store ไม่ได้

**อ้างอิง:**
- [React — Rendering Lists / Rules of keys](https://react.dev/learn/rendering-lists#rules-of-keys)
- [React — `<input>` checkbox แบบ controlled](https://react.dev/reference/react-dom/components/input#controlling-a-checkbox-with-a-state-variable)
- [React — `style` prop และการเติม px อัตโนมัติ](https://react.dev/reference/react-dom/components/common#applying-css-styles)

**ลองต่อยอด:** ลองเปลี่ยน `id={\`meal_${index}\`}` เป็น `id="meal"` ทั้ง 4 อัน แล้วคลิกที่ชื่อ "Dinner" ดู — จะเห็นว่า checkbox ของ Breakfast ติ๊กแทน

---

## [2026-09-21] Conference Event Planner — ทำไม build ผ่านแต่แอปพังตอน runtime

**Stack:** JavaScript / React / Vite / Redux Toolkit

**Concept:** ขอบเขตการตรวจจับของ bundler vs. linter vs. runtime — และ action creator รับ argument เดียว

**อธิบาย:**

**1. `vite build` ไม่ได้ตรวจว่าตัวแปรมีจริงหรือไม่**
- `ConferenceEvent.jsx:251` ส่ง `totalCosts={totalCosts}` แต่ **ไม่มีการประกาศ `totalCosts` ที่ไหนเลยในไฟล์** — ทั้ง `const`, parameter หรือ import
- esbuild/Rollup มองตัวระบุที่ไม่รู้จักว่าเป็น **global reference** (เช่น `window`, `document`) แล้วปล่อยผ่าน เพราะ bundler ไม่รู้ว่าตอน runtime จะมี global ตัวนั้นหรือไม่ → **build สำเร็จ 100%** แต่ระเบิดตอนเรนเดอร์
- การ **อ่าน** ตัวแปรที่ไม่ได้ประกาศ throw `ReferenceError` เสมอ ไม่ว่าจะ strict mode หรือไม่ (ที่ต่างกันคือตอน **assign**: sloppy mode จะสร้าง global ให้เงียบ ๆ ส่วน strict mode throw)
- เครื่องมือที่จับเคสนี้ได้คือ **ESLint กฎ `no-undef`** — นี่คือเหตุผลที่โปรเจกต์มี `npm run lint` แยกจาก `npm run build` ไม่ใช่ของประดับ
- **บทเรียน:** "build ผ่าน" ≠ "โค้ดถูก" — build ตอบแค่ว่า *แปลงเป็น bundle ได้ไหม* ไม่ได้ตอบว่า *ตรรกะถูกไหม* ลำดับที่ควรทำคือ lint → build → ทดสอบจริงในเบราว์เซอร์

**2. Error นี้เป็น conditional — ซ่อนตัวได้**
- `totalCosts` อยู่ในสาขา `showItems === true` ของ ternary ที่ `ConferenceEvent.jsx:248` → JSX ฝั่งนั้นจะถูก **evaluate ก็ต่อเมื่อเงื่อนไขเป็นจริง** ดังนั้นหน้าแรกโหลดปกติ แล้วค่อยจอขาวตอนกด "Show Details"
- เป็นตัวอย่างที่ดีว่าทำไม smoke test แค่ "เปิดหน้าแรกแล้วไม่พัง" ถึงไม่พอ ต้องเดินให้ครบทุก branch ของ UI

**3. Action creator ของ RTK รับ argument เดียวเท่านั้น**
- `handleMealSelection` เขียน `dispatch(toggleMealSelection(index, newNumberOfPeople))` — **argument ตัวที่สองถูกทิ้งทันที** action ที่วิ่งเข้า store คือ `{ type, payload: index }` เหมือนเดิม
- ถ้าต้องส่งหลายค่าจริง ๆ มี 2 ทาง: รวมเป็น object เดียว `toggleMealSelection({ index, people })` หรือใช้ **`prepare` callback** ใน `createSlice` ซึ่งรับ argument ได้หลายตัวแล้วประกอบเป็น payload เดียว
- อันตรายของเคสนี้คือ **มันไม่ error** — โค้ดดูเหมือนทำงาน แต่ค่าที่ตั้งใจส่งหายไปเงียบ ๆ

**4. เงื่อนไขที่เป็น dead branch**
- `if (item.selected && item.type === "mealForPeople")` — แต่ `mealsSlice.js` initialState มีแค่ `{ name, cost, selected }` **ไม่มี field `type`** → `item.type` เป็น `undefined` เสมอ → เงื่อนไขเป็น false ตลอด → ตกไปที่ `else` ทุกครั้ง
- บังเอิญว่า `else` คือพฤติกรรมที่ถูกต้องสำหรับแล็บนี้ โค้ดจึง "ทำงานได้" ทั้งที่ตรรกะครึ่งบนไม่เคยถูกใช้ — เป็น dead code ที่ควรลบหรือเติม field `type` ให้ครบ

**ทางเลือกที่พิจารณา / Trade-off:**
- **คำนวณ `totalCosts` ใน component** (ขยาย `calculateTotalCost` ให้รองรับ `"av"`/`"meals"` แล้วรวมเป็น object): ตรงไปตรงมา เหมาะกับแล็บ แต่คำนวณใหม่ทุก render
- **ใช้ memoized selector (`createSelector`)**: คำนวณซ้ำเฉพาะตอน state ที่เกี่ยวข้องเปลี่ยน เป็นแนวที่ scale ได้ แต่ over-engineering สำหรับ array 4-5 ตัว
- **เก็บยอดรวมไว้ใน store**: ผิดหลัก — ยอดรวมเป็น **derived state** ควรคำนวณจากแหล่งความจริง ไม่ใช่เก็บซ้ำแล้วเสี่ยง sync ไม่ตรง

**อ้างอิง:**
- [MDN — ReferenceError: "x" is not defined](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Not_defined)
- [Redux Toolkit — createSlice (prepare callback)](https://redux-toolkit.js.org/api/createSlice)
- [ESLint — `no-undef`](https://eslint.org/docs/latest/rules/no-undef)
- [Redux — Deriving Data with Selectors](https://redux.js.org/usage/deriving-data-selectors)

**ลองต่อยอด:** รัน `npm run lint` ตอนนี้เลย แล้วดูว่า `no-undef` ชี้บรรทัดไหนบ้าง — จะเห็นว่า linter จับ `totalCosts` ได้ตั้งแต่ก่อน build

---

## [2026-09-21] Conference Event Planner — Task 5.9: self-import และ derived state

**Stack:** JavaScript / React / Redux Toolkit

**Concept:** import อยู่ฝั่ง "ผู้ใช้" ไม่ใช่ฝั่ง "เจ้าของ" + การคำนวณ derived state

**อธิบาย:**

**1. บั๊กตัวจริงที่ทำให้ deploy ไม่ขึ้น — ไฟล์ import ตัวเอง**
- `mealsSlice.js` มี `import { toggleMealSelection } from "./mealsSlice";` อยู่บรรทัด 3 คือไฟล์ **import ตัวมันเอง** แล้วชนกับ `export const { toggleMealSelection } = mealsSlice.actions;` บรรทัดท้าย
- ผลคือ `SyntaxError: Identifier 'toggleMealSelection' has already been declared` — ใน ES Module ชื่อที่ import เข้ามาเป็น **binding ใน module scope** เหมือน `const` จึงประกาศซ้ำในไฟล์เดียวกันไม่ได้
- อันนี้ต่างจาก `totalCosts` ตรงที่เป็น **parse error → build พังทันที** ไม่ใช่ runtime error ที่ซ่อนตัว
- **หลักที่ต้องจำ:** `mealsSlice.js` คือ **ผู้ผลิต** (producer) ของ `toggleMealSelection` — มัน `export` ออกไป ไม่มีเหตุผลต้อง `import` กลับเข้ามา ส่วน `ConferenceEvent.jsx` คือ **ผู้บริโภค** (consumer) — ที่นั่นต่างหากที่ต้องมี import
- เวลาแล็บบอกว่า "make sure you have imported X" ให้ถามตัวเองก่อนเสมอว่า **ไฟล์ไหนคือคนใช้** ไม่ใช่ paste ลงไฟล์ที่เปิดค้างอยู่

**2. `calculateTotalCost(section)` — derived state ที่คำนวณสด ไม่เก็บใน store**
- ยอดรวมเป็น **derived state** (ข้อมูลที่คำนวณจากข้อมูลอื่นได้ 100%) หลักของ Redux คือ **อย่าเก็บสิ่งที่คำนวณได้** ลง store เพราะจะเกิด 2 แหล่งความจริงที่มีโอกาส sync ไม่ตรงกัน
- `venue`/`av` ใช้สูตร `cost × quantity` ส่วน `meals` ใช้ `cost × numberOfPeople` เพราะโมเดลข้อมูลต่างกัน — meals ไม่มี `quantity` มีแค่ `selected` (boolean) แล้วคูณด้วยจำนวนคนแทน จึงต้อง guard ด้วย `if (item.selected)` ก่อน
- `numberOfPeople` เป็น `useState` ใน component (ไม่ได้อยู่ใน Redux) แต่ถูกใช้ในการคำนวณได้ปกติ — **derived state ผสมข้อมูลจาก local state กับ store ได้** ไม่ผิดหลักอะไร
- ค่าที่คำนวณแล้วต้องเอาไปแสดงด้วย ไม่งั้นเป็น dead variable — จึงผูก `avTotalCost` / `mealsTotalCost` เข้ากับ `<div className="total_cost">` ของแต่ละ section

**3. ยืนยันบทเรียนเมื่อวาน**
- รันจริงแล้วได้ผลตามที่วิเคราะห์ไว้เป๊ะ: `npm run build` **สำเร็จ** ทั้งที่ `npm run lint` ยังฟ้อง `no-undef` ที่ `totalCosts` → bundler กับ linter ตรวจคนละอย่างจริง ๆ

**ทางเลือกที่พิจารณา / Trade-off:**
- **ฟังก์ชันเดียวรับ `section` เป็น string** (ที่ใช้อยู่ ตามแล็บ): เขียนสั้น แต่ string เป็น magic value ถ้าพิมพ์ผิดเป็น `"meal"` จะได้ `0` เงียบ ๆ ไม่มี error
- **แยกเป็น 3 ฟังก์ชัน** (`calcVenue`, `calcAv`, `calcMeals`): พิมพ์ผิดไม่ได้เพราะ JS จะ throw ทันที และแต่ละตัวอ่านง่ายกว่า — เป็นแบบที่ดีกว่าถ้าเขียนโปรดักชัน
- **`createSelector` (Reselect)**: memoize ได้ คำนวณซ้ำเฉพาะตอน input เปลี่ยน แต่ array 4-5 ตัวไม่คุ้ม

**อ้างอิง:**
- [MDN — JavaScript modules (import / export bindings)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
- [MDN — `import` declaration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import)
- [Redux — Deriving Data with Selectors](https://redux.js.org/usage/deriving-data-selectors)
- [Redux Style Guide — Keep State Minimal and Derive Additional Values](https://redux.js.org/style-guide/#keep-state-minimal-and-derive-additional-values)

**ลองต่อยอด:** ลองพิมพ์ `calculateTotalCost("meal")` (ตกตัว s) ดู — จะได้ `0` โดยไม่มี error เลย นี่คือราคาของการใช้ magic string ลองคิดว่าจะกันยังไง (constant object? TypeScript union type?)

**addendum (1) — `${...}` ใน JSX ไม่ใช่ template literal:** `<div>Total Cost: ${mealsTotalCost}</div>` ดูเหมือน template literal แต่ไม่ใช่ — มันคือ **ตัวอักษร `$` ธรรมดา** ตามด้วย **JSX expression container `{...}`** ซึ่งเป็นคนละไวยากรณ์กันโดยสิ้นเชิง template literal ต้องอยู่ใน backtick (`` `Total Cost: ${x}` ``) เท่านั้น ที่สับสนกันบ่อยเพราะผลลัพธ์ออกมาเหมือนกันพอดี ทดสอบได้โดยลบ `$` ออก — ตัวเลขยังแสดงปกติ เพราะ `{}` ทำงานของมันเองอยู่แล้ว อ้างอิง: [React — JSX with curly braces](https://react.dev/learn/javascript-in-jsx-with-curly-braces), [MDN — Template literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)

---

## [2026-09-21] Conference Event Planner — Deploy: content hashing กับ CDN cache

**Stack:** Vite / GitHub Pages (static hosting)

**Concept:** ทำไมไฟล์ asset มี hash ต่อท้าย และทำไม `index.html` ถึงค้าง cache

**อธิบาย:**
- หลัง deploy สำเร็จ (Pages build status = `built`, ไม่มี error) แต่ยิง request ไปหน้าเว็บกลับยังได้ `index-accJ1cMr.js` ตัวเก่า — พอยิงซ้ำแบบ `Cache-Control: no-cache` ถึงได้ `index-CxiW6x_X.js` ตัวใหม่ **ปัญหาอยู่ที่ CDN cache ไม่ใช่ deploy ล้มเหลว**
- Vite ตั้งชื่อไฟล์ build เป็น `index-<hash>.js` โดย hash คำนวณจาก **เนื้อหาไฟล์** → เนื้อหาเปลี่ยนเมื่อไหร่ ชื่อไฟล์เปลี่ยนตาม นี่คือเทคนิค **cache busting**: ตั้ง cache ของ asset ไว้ยาวมาก (1 ปี) ได้อย่างปลอดภัย เพราะไฟล์ใหม่ = URL ใหม่เสมอ ไม่มีทางได้ของเก่าผิดตัว
- แต่ `index.html` **มี hash ไม่ได้** เพราะเป็นจุดเข้าที่ URL ต้องคงที่ → มันจึงเป็นไฟล์เดียวที่ต้องพึ่ง cache header และเป็นตัวที่ค้างบ่อยที่สุด GitHub Pages ตั้ง `max-age` ของ HTML ไว้สั้น (ระดับนาที) รอสักพักหรือ hard refresh (Ctrl+F5) ก็หาย
- **วิธีแยกแยะว่า deploy พังจริงหรือแค่ cache:** เช็ก 3 ชั้นตามลำดับ — (1) `git ls-remote origin refs/heads/gh-pages` sha เปลี่ยนไหม (2) `gh api .../pages/builds/latest` status เป็น `built` และ commit ตรงไหม (3) ค่อยดูหน้าเว็บ ถ้า 2 ชั้นแรกผ่านแล้วชั้น 3 ยังเก่า = cache แน่นอน

**ข้อควรระวังด้าน security:**
- `gh-pages -d dist` publish **ทุกไฟล์ใน `dist/`** ขึ้นเว็บสาธารณะ — อะไรที่หลุดเข้า `dist/` หรือ `public/` จะอ่านได้จากอินเทอร์เน็ตทันที **ห้ามวาง `.env`, service account key หรือ credential ใด ๆ ใน `public/`** เด็ดขาด (Vite copy ทุกอย่างใน `public/` ไป `dist/` ตรง ๆ โดยไม่ผ่าน bundler)
- อีกข้อ: ตัวแปรที่ขึ้นต้นด้วย `VITE_` จะถูก **ฝังลงไฟล์ JS ที่ผู้ใช้โหลดได้** ไม่ใช่ความลับ — ใช้ได้เฉพาะค่าที่เปิดเผยได้ (เช่น Firebase web config) ไม่ใช่ secret key
- branch `gh-pages` ตอนนี้มี `.eslintrc.cjs` กับ `.gitignore` ค้างอยู่ (ไม่ได้อยู่ใน `dist/` แล้ว) เพราะ glob ลบไฟล์เก่าของ gh-pages ไม่แตะ dotfile โดยดีฟอลต์ — ไม่อันตรายแต่ควรล้าง

**ทางเลือกที่พิจารณา / Trade-off:**
- **`gh-pages` CLI** (ที่ใช้อยู่): ง่าย ไม่ต้องตั้งค่าอะไร แต่ deploy จากเครื่องตัวเอง → ของที่ขึ้นเว็บอาจไม่ตรงกับโค้ดที่ commit ไว้ และคนอื่นทำซ้ำไม่ได้
- **GitHub Actions**: deploy อัตโนมัติจาก commit บน `main` → ของที่ขึ้นเว็บตรงกับ git เสมอ ตรวจสอบย้อนหลังได้ เป็นมาตรฐานของงานจริง แต่ต้องเขียน workflow

**อ้างอิง:**
- [Vite — Building for Production (asset hashing)](https://vite.dev/guide/build.html)
- [Vite — Env Variables and Modes (คำเตือนเรื่อง `VITE_` prefix)](https://vite.dev/guide/env-and-mode.html)
- [Vite — The `public` Directory](https://vite.dev/guide/assets.html#the-public-directory)
- [GitHub Docs — About GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages)
- [MDN — HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)

**ลองต่อยอด:** แก้ CSS สักบรรทัดแล้ว build ใหม่ — จะเห็นว่า hash ของ `.css` เปลี่ยนแต่ `.js` ไม่เปลี่ยน เพราะ hash ผูกกับเนื้อหาของแต่ละไฟล์แยกกัน

---

## [2026-09-21] Conference Event Planner — Task 6: ตารางสรุป และ render prop

**Stack:** JavaScript / React

**Concept:** ส่ง component เป็น prop (render prop), spread + tagging, และกับดักของ `&&` ใน JSX

**อธิบาย:**

**1. ปิดบั๊ก `totalCosts` ที่ค้างมา 2 วัน**
- พอประกาศ `const totalCosts = { venue, av, meals }` แล้ว `no-undef` ที่ `ConferenceEvent.jsx` **หายไปจากผล lint ทันที** — ยืนยันว่า error ที่ linter ชี้ตรงกับ ReferenceError ที่จะเกิดจริงตอน runtime ไม่ใช่การเตือนเกินจริง
- ตัว object นี้ต้องวาง **หลัง** `venueTotalCost`/`avTotalCost`/`mealsTotalCost` เพราะ `const` มี **Temporal Dead Zone** — อ้างถึงก่อนบรรทัดที่ประกาศจะ throw ไม่เหมือน `var` ที่ได้ `undefined` เงียบ ๆ หรือ function declaration ที่ hoist ขึ้นไปทั้งตัว

**2. Render prop — ส่ง "วิธีเรนเดอร์" เป็น prop**
- `ItemsDisplay={() => <ItemsDisplay items={items} />}` ไม่ได้ส่ง *ข้อมูล* แต่ส่ง **ฟังก์ชันที่คืน JSX** ไปให้ลูก แล้วลูกเป็นคนตัดสินใจว่าจะเรียกตอนไหน/วางตรงไหน เรียกว่า **render prop**
- ข้อดีคือ `TotalCost` ไม่ต้องรู้จัก `items` หรือ `venueItems` เลย — มันแค่รู้ว่า "มีอะไรบางอย่างให้เรนเดอร์" จึงเอาไปใช้ซ้ำกับข้อมูลชุดอื่นได้ (inversion of control)
- ต้องห่อด้วย arrow function `() => <ItemsDisplay ... />` ไม่ใช่ `<ItemsDisplay items={items} />` ตรง ๆ เพราะ prop นี้ถูกประกาศให้เป็น **ฟังก์ชัน** ที่ลูกจะเรียกเอง
- สมัยใหม่นิยมใช้ **`children` prop** แทน render prop ในเคสง่าย ๆ แบบนี้ (อ่านง่ายกว่า) ส่วน custom hooks มาแทน render prop ในเคสที่แชร์ *ตรรกะ* ไม่ใช่ *UI*

**3. Spread + tagging เพื่อรวมข้อมูลต่างชนิด**
- `items.push({ ...item, type: "venue" })` คือ copy ทุก field ของ item แล้ว **เติม `type` เข้าไปเป็นป้ายกำกับ** ทำให้ array เดียวเก็บของ 3 ชนิดได้ แล้วค่อยแยกพฤติกรรมตอนเรนเดอร์ด้วย `item.type === "meals"`
- สำคัญ: spread สร้าง **object ใหม่** ไม่ได้แก้ของเดิมใน Redux store — ถ้าเขียน `item.type = "venue"` ตรง ๆ จะเป็นการ mutate state ที่ Redux ห้าม (และ middleware ของ `configureStore` จะจับได้ใน dev)
- นี่คือ **shallow copy** — ถ้า item มี object ซ้อนข้างใน จะยังแชร์ reference เดิมอยู่ เคสนี้ field เป็น primitive ล้วนจึงปลอดภัย

**4. กับดัก `&&` ใน JSX ที่ควรรู้**
- `{items.length === 0 && <p>No items selected</p>}` **ปลอดภัย** เพราะฝั่งซ้ายเป็น boolean
- แต่ถ้าเขียน `{items.length && <p>...</p>}` เมื่อ array ว่าง `items.length` เป็น `0` → JSX **เรนเดอร์เลข `0` ออกมาบนหน้าจอ** เพราะ `0` ไม่ใช่ค่าที่ React ข้าม (React ข้ามเฉพาะ `false`, `null`, `undefined`) เป็นบั๊กคลาสสิกที่เจอบ่อยมาก
- วิธีกัน: บังคับให้เป็น boolean เสมอ (`items.length > 0 &&`) หรือใช้ ternary

**5. ESLint เตือนใหม่: `react/prop-types`**
- `ItemsDisplay` รับ prop `items` แต่ไม่ได้ประกาศชนิดไว้ → `eslint-plugin-react` ฟ้อง 3 จุด เป็น **การเตือนเรื่องสัญญาระหว่าง component** ไม่ใช่บั๊ก โค้ดรันได้ปกติ
- ทางแก้: ติดตั้ง `prop-types` แล้วประกาศ `ItemsDisplay.propTypes = { items: PropTypes.array.isRequired }`, หรือปิดกฎ, หรือ **ย้ายไป TypeScript** ซึ่งตรวจตั้งแต่ compile time แทนที่จะรอ runtime (ทีมส่วนใหญ่เลือกทางนี้แล้ว)

**ทางเลือกที่พิจารณา / Trade-off:**
- **Render prop** (ที่แล็บใช้): ยืดหยุ่น ลูกคุมตำแหน่งเรนเดอร์ได้ | **`children`**: อ่านง่ายกว่าสำหรับเคสเดียว | **ส่ง `items` เป็น data ตรง ๆ**: ง่ายสุดแต่ `TotalCost` ต้องรู้รูปร่างข้อมูล ผูกกันแน่นขึ้น
- **`key={index}`**: ยังใช้ได้เพราะ list สร้างใหม่ทุกครั้งและไม่มี local state ใน row — แต่ `key={`${item.type}-${item.name}`}` จะสื่อความหมายกว่า

**อ้างอิง:**
- [React — Passing props to a component](https://react.dev/learn/passing-props-to-a-component)
- [React — Conditional rendering (กับดัก `&&` กับเลข 0)](https://react.dev/learn/conditional-rendering#logical-and-operator-)
- [React (legacy docs) — Render Props](https://legacy.reactjs.org/docs/render-props.html)
- [MDN — Spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [MDN — `let`/`const` และ Temporal Dead Zone](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz)

**ลองต่อยอด:** เปลี่ยน `{items.length === 0 && ...}` เป็น `{items.length && ...}` แล้วกด Show Details ตอนยังไม่เลือกอะไร — จะเห็นเลข `0` โผล่บนหน้าจอ เป็นการพิสูจน์กับดักข้อ 4 ด้วยตาตัวเอง

---

## [2026-09-21] Conference Event Planner — Task 7: TotalCost และกฎตัวพิมพ์ใหญ่ของ JSX

**Stack:** JavaScript / React

**Concept:** ทำไม `<ItemsDisplay />` ถึงเรียกฟังก์ชันที่รับมาทาง prop ได้

**อธิบาย:**

**1. กฎสำคัญ: JSX ตัดสินจาก "ตัวพิมพ์ใหญ่/เล็ก" ของชื่อ**
- `<ItemsDisplay />` ใน `TotalCost.jsx` ไม่ได้อ้างถึง component ที่ import มา — มันอ้างถึง **prop ชื่อ `ItemsDisplay`** ที่แม่ส่งมา ซึ่งค่าจริงคือ `() => <ItemsDisplay items={items} />`
- ที่มันทำงานได้เพราะ JSX แปลงตามกฎนี้: **ชื่อขึ้นต้นด้วยตัวพิมพ์ใหญ่ → ถือเป็นตัวแปรใน scope แล้วเรียกเป็น component** ส่วน **ชื่อขึ้นต้นด้วยตัวพิมพ์เล็ก → ถือเป็น HTML tag (string)**
- ดังนั้นถ้าเปลี่ยนชื่อ prop เป็น `itemsDisplay` (ตัวเล็ก) แล้วเขียน `<itemsDisplay />` React จะพยายามสร้าง DOM element ชื่อ `<itemsdisplay>` แทน — ไม่ error แต่หน้าจอว่างเปล่า เป็นบั๊กที่หาสาเหตุยากมาก
- สรุปสายการทำงานเต็ม: `ConferenceEvent` สร้าง `items` → ห่อเป็น arrow function ส่งเป็น prop → `TotalCost` เรียกด้วย `<ItemsDisplay />` → React เรียกฟังก์ชันนั้น → ได้ `<ItemsDisplay items={items} />` ของ `ConferenceEvent` กลับมาเรนเดอร์ (ปิดวงจร render prop จาก Task 6)

**2. `total_amount` คำนวณสดจาก props ทุก render**
- ไม่ต้อง `useState` หรือ `useEffect` เลย — เป็น **derived value** จาก props ล้วน ๆ การเก็บลง state จะทำให้มีสองแหล่งความจริงและต้องเขียน effect คอย sync ซึ่งเป็น anti-pattern ที่ React เตือนไว้ตรง ๆ
- ก็เลยเป็นเหตุผลว่าทำไม `useState`/`useEffect` ที่ import มาตั้งแต่ต้นไฟล์ถึงไม่ถูกใช้ (ESLint ฟ้อง `no-unused-vars`) — แล็บให้ import ไว้เผื่อ แต่โจทย์จริงไม่ต้องใช้ ลบได้ปลอดภัย

**3. บั๊ก HTML ที่ซ่อนอยู่ในโค้ดแล็บ: `<h3>` ใน `<p>`**
- `<p className="preheading"><h3>Total cost for the event</h3></p>` เป็น **invalid nesting** — สเปก HTML ห้าม `<p>` มี block-level element ข้างใน เบราว์เซอร์จะ **auto-close `<p>` ก่อน `<h3>`** ทำให้ DOM จริงกลายเป็น `<p></p><h3>...</h3><p></p>`
- ผลคือ `.pricing-app .preheading` (font 25px, uppercase, dosis) ไปลงที่ `<p>` เปล่า ๆ ส่วน `<h3>` หลุดออกมาใช้สไตล์ default → หน้าตาไม่ตรงกับที่ CSS ตั้งใจ
- React จะ log `validateDOMNesting(...): <h3> cannot appear as a descendant of <p>` ใน console ด้วย
- แก้ได้โดยเลือกอย่างใดอย่างหนึ่ง: `<p className="preheading">Total cost for the event</p>` หรือ `<h3 className="preheading">Total cost for the event</h3>`

**ทางเลือกที่พิจารณา / Trade-off:**
- **`snake_case` (`total_amount`) ตามแล็บ**: ขัดกับ convention ของ JS ที่ใช้ `camelCase` (`totalAmount`) — ไม่ผิดแต่ไม่เข้าพวกกับตัวแปรอื่นในโปรเจกต์อย่าง `venueTotalCost` ถ้าทำงานจริงควรเลือกให้สม่ำเสมอทั้ง codebase
- **คำนวณ `total_amount` ในลูก (ที่ใช้อยู่)**: ลูกรู้วิธีรวมเอง แม่ส่งแค่ข้อมูลดิบ | **คำนวณในแม่แล้วส่งตัวเลขมา**: ลูกโง่ลง reusable น้อยลง แต่ tracing ง่ายกว่า

**อ้างอิง:**
- [React — Writing markup with JSX (กฎตัวพิมพ์ใหญ่ของชื่อ component)](https://react.dev/learn/writing-markup-with-jsx)
- [React — Your First Component: ชื่อ component ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่](https://react.dev/learn/your-first-component#step-2-define-the-function)
- [React — You Might Not Need an Effect (อย่าเก็บ derived value ลง state)](https://react.dev/learn/you-might-not-need-an-effect)
- [HTML Standard — The `p` element (content model: phrasing content เท่านั้น)](https://html.spec.whatwg.org/multipage/grouping-content.html#the-p-element)

**ลองต่อยอด:** เปิด DevTools → Elements ตอนกด Show Details แล้วดู DOM ของ `.header` จะเห็นว่า `<h3>` ไม่ได้อยู่ใน `<p>` อย่างที่เขียนไว้ในโค้ด — เป็นตัวอย่างว่าเบราว์เซอร์ "ซ่อม" HTML ผิดกฎให้เราโดยไม่บอก
