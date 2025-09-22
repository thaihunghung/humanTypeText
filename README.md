# humanTypeText
`async function humanTypeText(page, selector = '', text, options = {}) {
    const {
        clearBefore = true,
        minDelay = 80,
        maxDelay = 200,
        pressEnterAfter = false,
        validateElement = true,
        naturalPauses = true,
        language = 'auto',
        inputMethod = 'auto'
    } = options;

    const languageConfigs = {
        'en': {
            name: 'English',
            wordBreakPattern: /\S+/g,
            commonPunctuation: /[.,!?;:""'()\-]/,
            complexCharDelay: 1.0,
            needsIME: false
        },
        'vi': {
            name: 'Tiếng Việt',
            wordBreakPattern: /\S+/g,
            commonPunctuation: /[.,!?;:""'()]/,
            complexCharDelay: 1.4,
            needsIME: false,
            needsTelex: true,
            accentedChars: /[àáảãạăắằẳẵặâấầẩẫậèéẻẽẹêếềểễệìíỉĩịòóỏõọôốồổỗộơớờởỡợùúủũụưứừửữựỳýỷỹỵđ]/gi,
            telexMap: {
                'aa': 'â', 'aw': 'ă', 'ee': 'ê', 'oo': 'ô', 'ow': 'ơ', 'uw': 'ư', 'dd': 'đ',
                'af': 'à', 'as': 'á', 'ar': 'ả', 'ax': 'ã', 'aj': 'ạ'
            }
        },
        'zh': {
            name: '中文 (简体)',
            wordBreakPattern: /[\u4e00-\u9fff]|[a-zA-Z0-9]+|\s+|[^\u4e00-\u9fff\w\s]/g,
            commonPunctuation: /[，。！？；：""''（）【】《》]/,
            complexCharDelay: 2.2,
            needsIME: true,
            pinyinMap: {
                '你': 'ni', '好': 'hao', '我': 'wo', '是': 'shi', '的': 'de', '了': 'le',
                '在': 'zai', '有': 'you', '他': 'ta', '这': 'zhe', '中': 'zhong', '人': 'ren',
                '一': 'yi', '上': 'shang', '也': 'ye', '很': 'hen', '到': 'dao', '说': 'shuo',
                '繁': 'fan', '荣': 'rong', '富': 'fu', '足': 'zu'
            }
        }
    };

    async function detectLanguage(text) {
        if (language !== 'auto') return languageConfigs[language] || languageConfigs['en'];
        const chineseChars = (text.match(/[\u4e00-\u9fff]/g) || []).length;
        const vietnameseChars = (text.match(/[àáảãạăắằẳẵặâấầẩẫậèéẻẽẹêếềểễệìíỉĩịòóỏõọôốồổỗộơớờởỡợùúủũụưứừửữựỳýỷỹỵđ]/gi) || []).length;
        const totalChars = text.length || 1; // Avoid division by zero
        if (chineseChars / totalChars > 0.3) return languageConfigs['zh'];
        if (vietnameseChars / totalChars > 0.08) return languageConfigs['vi'];
        if (text.match(/[a-zA-Z\s.,!?;:"'()-]/g)?.length / totalChars > 0.5) return languageConfigs['en'];
        return languageConfigs['en']; // Default to English
    }

    const config = await detectLanguage(text);
    console.log(`🎯 Detected: ${config.name}`);

    if (selector) {
        try {
            const element = page.locator(selector);
            if (validateElement) {
                await element.waitFor({ state: 'visible', timeout: 10000 });
            }

            const elementBox = await element.boundingBox();
            if (elementBox) {
                await humanMouseMove(page, elementBox.x + elementBox.width / 2, elementBox.y + elementBox.height / 2);
                await page.waitForTimeout(shortDelay(200, 600));
            }

            await element.click();
            await page.waitForTimeout(humanDelay(300, 800));

            // Verify focus
            const isFocused = await element.evaluate(el => el === document.activeElement);
            if (!isFocused) {
                console.warn('⚠️ Element not focused, retrying...');
                await element.click();
                await page.waitForTimeout(humanDelay(300, 800));
            }

            if (clearBefore) {
                await page.keyboard.press('Control+KeyA');
                await page.waitForTimeout(shortDelay(100, 300));
                await page.keyboard.press('Delete');
                await page.waitForTimeout(shortDelay(200, 500));
            }
        } catch (error) {
            console.warn(`⚠️ Cannot interact with selector: ${selector}`, error.message);
            throw error;
        }
    } else {
        console.log('📝 No selector provided, typing into active element...');
    }

    if (!text || typeof text !== 'string') {
        console.warn('⚠️ Text must be string and not empty');
        return;
    }

    // Check if IME is available
    const imeAvailable = await page.evaluate(() => !!window.InputMethodContext || !!document.activeElement?.setAttribute('lang', 'zh-CN'));
    console.log(`🎌 IME available: ${imeAvailable}`);

    const segments = text.match(config.wordBreakPattern) || [text];
    const separators = text.split(config.wordBreakPattern).filter(s => s);

    for (let segIndex = 0; segIndex < segments.length; segIndex++) {
        const segment = segments[segIndex];

        if (naturalPauses && Math.random() < 0.3) {
            await page.waitForTimeout(humanDelay(500, 2000));
        }

        if (config.needsIME && inputMethod !== 'direct' && imeAvailable) {
            console.log('🎌 Using IME input for Simplified Chinese...');
            await typeWithIME(page, segment, config, minDelay, maxDelay);
        } else if (config.needsTelex && inputMethod !== 'direct') {
            console.log('🇻🇳 Using Telex input...');
            await typeWithTelex(page, segment, config, minDelay, maxDelay);
        } else {
            console.log('📝 Using direct input...');
            await typeDirect(page, segment, config, minDelay, maxDelay);
        }

        if (segIndex < segments.length - 1 && separators[segIndex]) {
            console.log(`Typing separator: ${separators[segIndex]}`);
            await page.keyboard.type(separators[segIndex]);
            if (config.commonPunctuation.test(separators[segIndex])) {
                await page.waitForTimeout(humanDelay(300, 800));
            }
        }
    }

    if (pressEnterAfter) {
        await page.waitForTimeout(humanDelay(500, 1500));
        await page.keyboard.press('Enter');
    }

    async function typeDirect(page, text, config, minDelay, maxDelay) {
        for (let i = 0; i < text.length; i++) {
            const char = text[i];
            if (naturalPauses && Math.random() < 0.05 && i > 0 && !/[\u4e00-\u9fff]/.test(char)) {
                const wrongChar = String.fromCharCode(char.charCodeAt(0) + (Math.random() > 0.5 ? 1 : -1));
                await page.keyboard.type(wrongChar);
                await page.waitForTimeout(shortDelay(100, 300));
                await page.keyboard.press('Backspace');
                await page.waitForTimeout(shortDelay(200, 500));
            }
            console.log(`Typing char: ${char}`);
            await page.keyboard.type(char);
            let delay = minDelay + Math.random() * (maxDelay - minDelay);
            if (config.commonPunctuation.test(char)) {
                delay *= config.complexCharDelay;
            }
            if (config.accentedChars?.test(char)) {
                delay *= config.complexCharDelay;
            }
            if (/[\u4e00-\u9fff]/.test(char)) {
                delay *= config.complexCharDelay;
            }
            await page.waitForTimeout(delay);
        }
    }

    async function typeWithIME(page, text, config, minDelay, maxDelay) {
        for (let char of text) {
            if (/[\u4e00-\u9fff]/.test(char)) {
                const pinyin = config.pinyinMap[char] || generatePinyin(char);
                console.log(`Typing pinyin '${pinyin}' for char '${char}'`);
                for (let p of pinyin) {
                    await page.keyboard.type(p);
                    await page.waitForTimeout(humanDelay(60, 150));
                }
                await page.waitForTimeout(humanDelay(200, 800));
                if (Math.random() < 0.3) {
                    const candidateNumber = Math.floor(Math.random() * 5) + 1;
                    await page.keyboard.press(`Digit${candidateNumber}`);
                } else {
                    await page.keyboard.press('Space');
                }
                await page.waitForTimeout(humanDelay(100, 400));
            } else {
                console.log(`Typing char: ${char}`);
                await page.keyboard.type(char);
                await page.waitForTimeout(humanDelay(minDelay, maxDelay) * config.complexCharDelay);
            }
        }
    }

    async function typeWithTelex(page, text, config, minDelay, maxDelay) {
        const telexSequence = convertToTelex(text, config);
        for (let char of telexSequence) {
            if (naturalPauses && Math.random() < 0.05) {
                const wrongAccent = { 'f': 's', 's': 'f', 'r': 'x', 'x': 'r', 'j': 's' }[char] || char;
                await page.keyboard.type(wrongAccent);
                await page.waitForTimeout(shortDelay(100, 300));
                await page.keyboard.press('Backspace');
                await page.waitForTimeout(shortDelay(200, 500));
            }
            console.log(`Typing char: ${char}`);
            await page.keyboard.type(char);
            let delay = minDelay + Math.random() * (maxDelay - minDelay);
            if (char.match(/[fsrxj]/)) {
                delay *= config.complexCharDelay;
            }
            await page.waitForTimeout(delay);
        }
    }

    function convertToTelex(word, config) {
        let result = word;
        for (const [telex, char] of Object.entries(config.telexMap)) {
            result = result.replace(new RegExp(telex, 'gi'), char);
        }
        return result;
    }

    function generatePinyin(char) {
        console.warn(`⚠️ No pinyin mapping for '${char}', using fallback 'pin'`);
        return 'pin'; // Replace with a pinyin library for production
    }

    function humanDelay(min, max) {
        return Math.floor(Math.random() * (max - min + 1)) + min;
    }

    function shortDelay(min, max) {
        return Math.floor(Math.random() * (max - min + 1)) + min;
    }

    async function humanMouseMove(page, x, y) {
        await page.mouse.move(x, y, { steps: Math.floor(Math.random() * 10) + 5 });
    }
}`
