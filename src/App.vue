<template>
    <div>
        <h1>{{ title }}</h1>
        <div class="rating-container">
            <div v-for="level in levels" :key="level.name" class="rating-row">
                <h2>{{ level.name }}</h2>
                <div
                    class="drop-zone"
                    @dragover.prevent
                    @drop="onDrop(level)"
                >
                    <div
                        v-for="item in level.items"
                        :key="item.id"
                        class="icon"
                        draggable="true"
                        @dragstart="onDragStart(item)"
                    >
                        {{ item.name }}
                    </div>
                </div>
            </div>
        </div>
        <div class="icons-container" @wheel="onWheel" @dragover.prevent @drop="onDrop(perLevel)">
            <div
                v-for="(item, index) in perLevel.items"
                :key="item.id"
                class="icon icon-fix-position"
                :style="item.fixPosition(index)"
                draggable="true"
                @dragstart="onDragStart(item)"
            >
                {{ item.name }}
            </div>
        </div>
        <button class="submit-button" @click="submitRatings">提交评级</button>
        <button class="reset-button" @click="resetRatings">重置</button>
    </div>
</template>

<script>
class Item {
    constructor(id, name, level = null) {
        this.id = id;
        this.name = name;
        this.level = level;
        level && level.appendItem(this); // 如果有级别，则添加到该级别
    }

    setLevel(level) {
        if (this.level) {
            this.level.removeItem(this); // 从当前级别移除
        }
        // 添加到新级别
        this.level = level;
        if (this.isInPerlevel()){
            level.insertItem(this);
        }
        else level.appendItem(this); 
    }

    isInPerlevel() {
        return this.level.name === '0'; // 需要修正位置的条件
    }

    fixPosition(index) {
        if (this.isInPerlevel()) {
            // 如果在待评级区，设置堆叠效果
            return {
                zIndex: this.level.items.length - index,
                top: `${index * 5}px`, 
                left: `${index * 5}px`,
            }
        }
        else return {};
    }
}

class Level {
    constructor(name) {
        this.name = name;
        this.items = [];
    }

    appendItem(item) {
        this.items.push(item);
    }

    insertItem(item) {
        this.items.unshift(item);
    }

    removeItem(item) {
        this.items = this.items.filter(i => i.id !== item.id);
    }
}

export default {
    name: 'App',
    data() {
        return {
            // 页面标题
            title: '评级功能',
            // 评级等级
            levels: {
                S: new Level('S'),
                A: new Level('A'),
                B: new Level('B'),
                C: new Level('C'),
                D: new Level('D'),
                E: new Level('E'),
            },
            // 备选区
            perLevel : new Level('0'),

            // 待评级对象
            items: [],

            // 当前拖拽的对象
            draggedItem: null,
        };
    },
    methods: {
        // 拖拽开始时记录拖拽的对象及其来源
        onDragStart(item) {
            this.draggedItem = item;
        },
        // 拖拽释放时处理对象的移动逻辑
        onDrop(level) {
            if (this.draggedItem) {
                // 设置拖拽对象的新级别
                this.draggedItem.setLevel(level);
                // 清空拖拽状态
                this.draggedItem = null;
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
                    const firstItem = this.perLevel.items.shift(); // 移除第一个元素
                    this.perLevel.items.push(firstItem); // 将其添加到数组末尾
                } else {
                    const lastItem = this.perLevel.items.pop();
                    this.perLevel.items.unshift(lastItem);
                }
            }
        },
        // 重置所有评级，将对象移回待评级区域
        resetRatings() {
            for(let i = this.items.length - 1; i >= 0; i--) {
                this.items[i].setLevel(this.perLevel);
            }
        },
    },
    mounted() {
        // 添加评级对象
        for (let i = 1; i <= 5; i++) {
            const item = new Item(i, `元素 ${i}`, this.perLevel);
            this.items.push(item);
        }
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
}

.icon-fix-position {
    position: absolute; /* 设置为绝对定位以支持堆叠 */
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
