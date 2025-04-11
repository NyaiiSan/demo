<template>
    <div>
        <h1>{{ title }}</h1>
        <div class="rating-container">
            <div v-for="level in levels" :key="level" class="rating-row">
                <h2>{{ level }}</h2>
                <div
                    class="drop-zone"
                    @dragover.prevent
                    @drop="onDrop(level)"
                >
                    <div
                        v-for="item in ratings[level]"
                        :key="item.id"
                        class="icon"
                        draggable="true"
                        @dragstart="onDragStart(item, level)"
                    >
                        {{ item.name }}
                    </div>
                </div>
            </div>
        </div>
        <div class="icons-container" @wheel="onWheel" @dragover.prevent @drop="onDrop('items')">
            <div
                v-for="(item, index) in items"
                :key="item.id"
                class="icon"
                :style="{ zIndex: items.length - index, top: `${index * 5}px`, left: `${index * 5}px` }"
                draggable="true"
                @dragstart="onDragStart(item, 'items')"
            >
                {{ item.name }}
            </div>
        </div>
        <button class="submit-button" @click="submitRatings">提交评级</button>
        <button class="reset-button" @click="resetRatings">重置</button>
    </div>
</template>

<script>
export default {
    name: 'App',
    data() {
        return {
            // 页面标题
            title: '评级功能',
            // 评级等级
            levels: ['S', 'A', 'B', 'C', 'D', 'E'],
            // 待评级的对象列表
            items: [
                { id: 1, name: '对象1' },
                { id: 2, name: '对象2' },
                { id: 3, name: '对象3' },
            ],
            // 各评级等级对应的对象
            ratings: {
                S: [],
                A: [],
                B: [],
                C: [],
                D: [],
                E: [],
            },
            // 当前拖拽的对象
            draggedItem: null,
            // 当前拖拽对象的来源
            draggedFrom: null,
        };
    },
    methods: {
        // 拖拽开始时记录拖拽的对象及其来源
        onDragStart(item, from = null) {
            this.draggedItem = item;
            this.draggedFrom = from;
        },
        // 拖拽释放时处理对象的移动逻辑
        onDrop(level) {
            if (this.draggedItem) {
                if (this.draggedFrom && this.draggedFrom !== level) {
                    // 如果从评级区域拖回待评级区域
                    if (level === 'items') {
                        this.items.push(this.draggedItem);
                        this.ratings[this.draggedFrom] = this.ratings[this.draggedFrom].filter(
                            (i) => i.id !== this.draggedItem.id
                        );
                    } else {
                        // 从一个评级区域拖到另一个评级区域
                        if (this.draggedFrom !== 'items') {
                            this.ratings[this.draggedFrom] = this.ratings[this.draggedFrom].filter(
                                (i) => i.id !== this.draggedItem.id
                            );
                        }
                        this.ratings[level].push(this.draggedItem);
                    }
                }

                // 如果从待评级区域拖到评级区域
                if (this.draggedFrom === 'items' && level !== 'items') {
                    this.items = this.items.filter((i) => i.id !== this.draggedItem.id);
                    this.ratings[level].push(this.draggedItem);
                }

                // 清空拖拽状态
                this.draggedItem = null;
                this.draggedFrom = null;
            }
        },
        // 提交评级结果到服务器
        async submitRatings() {
            try {
                const response = await fetch('/api/submit-ratings', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(this.ratings),
                });
                if (response.ok) {
                    alert('评级结果提交成功！');
                } else {
                    alert('提交失败，请重试。');
                }
            } catch (error) {
                console.error('提交出错:', error);
                alert('提交出错，请检查网络连接。');
            }
        },
        // 鼠标滚轮事件，用于调整待评级对象的顺序
        onWheel(event) {
            if (this.items.length > 1) {
                if (event.deltaY > 0) {
                    // 向下滚动，将第一个元素移到数组末尾
                    const firstItem = this.items.shift();
                    this.items.push(firstItem);
                } else {
                    // 向上滚动，将最后一个元素移到数组开头
                    const lastItem = this.items.pop();
                    this.items.unshift(lastItem);
                }
            }
        },
        // 重置所有评级，将对象移回待评级区域
        resetRatings() {
            // 将所有评级对象重置到待评级区域
            const allItems = Object.values(this.ratings).flat(); // 获取所有评级中的对象
            this.items = [...this.items, ...allItems]; // 合并到待评级区域
            this.levels.forEach((level) => {
                this.ratings[level] = []; // 清空每个评级级别
            });

            // 确保重置后待评级区的对象堆叠效果
            this.items = this.items.map((item, index) => ({
                ...item,
                zIndex: this.items.length - index, // 设置堆叠顺序
            }));
        },
    },
};
</script>

<style>
.rating-container {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.rating-row {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 300px; /* 设置最小宽度 */
}

.drop-zone {
    flex: 1;
    min-height: 50px;
    border: 1px dashed #ccc;
    display: flex;
    align-items: center;
    gap: 10px; /* 调整间距，确保对象不会堆叠 */
    flex-wrap: wrap; /* 支持多行排列 */
    padding: 5px;
    background-color: #fafafa; /* 添加背景色 */
    border-radius: 5px; /* 添加圆角 */
}

.icons-container {
    margin-top: 20px;
    position: relative; /* 设置为相对定位以支持堆叠 */
    width: 100px; /* 固定宽度以限制堆叠区域 */
    height: 100px; /* 固定高度以限制堆叠区域 */
    border: 2px dashed #007bff; /* 添加边框以区分待评级区域 */
    border-radius: 5px; /* 添加圆角 */
    padding: 5px; /* 添加内边距 */
    background-color: #f9f9f9; /* 添加背景色 */
}

.icon {
    width: 50px;
    height: 50px;
    background-color: #f0f0f0;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid #ccc;
    border-radius: 50%; /* 设置为圆角 */
    cursor: grab;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* 添加阴影 */
    transition: transform 0.2s, box-shadow 0.2s; /* 保留交互动画 */
    position: absolute; /* 确保待评级区对象堆叠 */
}

.icon:active {
    transform: scale(0.95); /* 点击时缩小 */
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2); /* 点击时阴影变化 */
}

h1 {
    text-align: center;
    font-size: 24px;
    color: #333;
    margin-bottom: 20px;
}

h2 {
    font-size: 18px;
    color: #555;
    margin: 0;
}

.submit-button {
    margin-top: 20px;
    padding: 10px 20px;
    font-size: 16px;
    color: #fff;
    background-color: #007bff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.submit-button:hover {
    background-color: #0056b3;
}

.reset-button {
    margin-top: 10px;
    padding: 10px 20px;
    font-size: 16px;
    color: #fff;
    background-color: #dc3545;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.reset-button:hover {
    background-color: #c82333;
}
</style>
