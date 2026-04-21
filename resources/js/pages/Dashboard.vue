<script setup>
import { Head, useForm, router } from '@inertiajs/vue3';
import { ref, computed, watch } from 'vue';
import {
    parseDate,
    DateFormatter,
    getLocalTimeZone,
} from '@internationalized/date';
import { toast } from 'vue-sonner';
import { dashboard } from '@/routes';

//UI Components
import {
    Accordion,
    AccordionContent,
    AccordionItem,
    AccordionTrigger,
} from '@/components/ui/accordion';
import {
    Dialog,
    DialogClose,
    DialogContent,
    DialogDescription,
    DialogFooter,
    DialogHeader,
    DialogTitle,
    DialogTrigger,
} from '@/components/ui/dialog';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import {
    Select,
    SelectContent,
    SelectItem,
    SelectTrigger,
    SelectValue,
} from '@/components/ui/select';
import { Calendar } from '@/components/ui/calendar';
import {
    Popover,
    PopoverContent,
    PopoverTrigger,
} from '@/components/ui/popover';

//Icons
import { Pencil, Trash2, CalendarClock } from 'lucide-vue-next';

//Page Layout
defineOptions({
    layout: {
        breadcrumbs: [
            {
                title: 'Dashboard',
                href: dashboard(),
            },
        ],
    },
});

//Props
const props = defineProps({
    tasks: {
        type: Array,
        default: () => [],
    },
});

//State
const form = useForm({
    title: '',
    description: '',
    due_date: '',
    is_completed: false,
});
const isOpen = ref(false); //Modal
const isEditMode = ref(false); //Edit Modal
const editingTaskId = ref(null);
const searchQuery = ref('');
const calendarDate = ref();
const df = new DateFormatter('en-US', { dateStyle: 'long' });

watch(calendarDate, (newDate) => {
    form.due_date = newDate ? newDate.toString() : '';
});

//Filter
const pendingTasks = computed(() => {
    return props.tasks.filter((task) => {
        const isMatch = task.title
            .toLowerCase()
            .includes(searchQuery.value.toLowerCase());
        return !task.is_completed && isMatch;
    });
});

const completedTasks = computed(() => {
    return props.tasks.filter((task) => {
        const isMatch = task.title
            .toLowerCase()
            .includes(searchQuery.value.toLowerCase());
        return task.is_completed && isMatch;
    });
});

//Methods
const submitTask = () => {
    //Store Task
    if (isEditMode.value) {
        form.put(`/tasks/${editingTaskId.value}`, {
            onSuccess: () => {
                toast.success('Task Updated', {
                    description: `"${form.title}" has been updated.`,
                });
                closeModal();
            },
        });
    } else {
        form.post('/tasks', {
            onSuccess: () => {
                toast.success('Task Created', {
                    description: `"${form.title}" has been added.`,
                });
                closeModal();
            },
        });
    }
};
const openEditModal = (task) => {
    //Update Task
    isEditMode.value = true;
    editingTaskId.value = task.id;

    form.title = task.title;
    form.description = task.description || '';
    form.due_date = task.due_date || '';
    form.is_completed = task.is_completed;

    calendarDate.value = task.due_date ? parseDate(task.due_date) : undefined;

    isOpen.value = true;
};
const closeModal = () => {
    isOpen.value = false;
    isEditMode.value = false;
    editingTaskId.value = null;
    calendarDate.value = undefined;
    form.reset();
};
const toggleTask = (event, task) => {
    //Update Task by Toggle
    event.preventDefault();

    const action = task.is_completed ? 'undo completion of' : 'complete';

    if (confirm(`Are you sure you want to ${action} this task?`)) {
        router.put(
            `/tasks/${task.id}`,
            {
                is_completed: !task.is_completed,
            },
            {
                preserveScroll: true,
                onSuccess: () => {
                    const statusText = !task.is_completed
                        ? 'Completed'
                        : 'Pending';
                    toast.success('Task Updated', {
                        description: `"${task.title}" has been moved to ${statusText}.`,
                    });
                },
            },
        );
    }
};
const deleteTask = (task) => {
    //Delete Task
    if (confirm(`Are you sure you want to delete "${task.title}"?`)) {
        router.delete(`/tasks/${task.id}`, {
            preserveScroll: true,
            onSuccess: () => {
                toast.success('Task Deleted', {
                    description: `"${task.title}" has been removed successfully.`,
                });
            },
        });
    }
};
</script>

<template>
    <Head title="Dashboard" />

    <div
        class="flex h-full flex-1 flex-col gap-4 overflow-x-auto rounded-xl p-4"
    >
        <div
            class="relative min-h-[100vh] flex-1 rounded-xl border border-sidebar-border/70 bg-white p-6 md:min-h-min dark:border-sidebar-border"
        >
            <div class="mb-6 flex items-center justify-between">
                <h2 class="text-xl font-bold">Task Overview</h2>
            </div>

            <div
                class="mb-6 flex flex-col items-center justify-between gap-4 sm:flex-row"
            >
                <div class="relative w-full max-w-sm">
                    <span
                        class="absolute inset-y-0 left-3 flex items-center text-gray-400"
                    >
                    </span>
                    <Input
                        v-model="searchQuery"
                        placeholder="Search tasks..."
                        class="pl-10"
                    />
                </div>

                <Dialog v-model:open="isOpen">
                    <DialogTrigger as-child>
                        <Button
                            variant="default"
                            @click="
                                isEditMode = false;
                                form.reset();
                            "
                        >
                            + Add Task
                        </Button>
                    </DialogTrigger>

                    <DialogContent>
                        <DialogHeader>
                            <DialogTitle>{{
                                isEditMode ? 'Edit Task' : 'Add New Task'
                            }}</DialogTitle>
                            <DialogDescription>
                                {{
                                    isEditMode
                                        ? 'Update your task details below.'
                                        : 'What you need to get done?'
                                }}
                            </DialogDescription>
                        </DialogHeader>

                        <form @submit.prevent="submitTask">
                            <div class="grid gap-4 py-4">
                                <Label for="task">Task Title</Label>
                                <Input
                                    id="task"
                                    v-model="form.title"
                                    required
                                />
                                <Label for="description"
                                    >Task Description (Optional)</Label
                                >
                                <Textarea
                                    placeholder="Type task description here"
                                    id="task"
                                    v-model="form.description"
                                />

                                <div class="grid grid-cols-2 gap-4">
                                    <div class="grid gap-2">
                                        <Label for="due_date">Due Date</Label>
                                        <Popover>
                                            <PopoverTrigger as-child>
                                                <Button
                                                    variant="outline"
                                                    :class="[
                                                        'w-full justify-start text-left font-normal',
                                                        !calendarDate &&
                                                            'text-muted-foreground',
                                                    ]"
                                                >
                                                    <CalendarClock
                                                        class="mr-2 h-4 w-4"
                                                    />
                                                    {{
                                                        calendarDate
                                                            ? df.format(
                                                                  calendarDate.toDate(
                                                                      getLocalTimeZone(),
                                                                  ),
                                                              )
                                                            : 'Pick a date'
                                                    }}
                                                </Button>
                                            </PopoverTrigger>
                                            <PopoverContent
                                                class="w-auto p-0"
                                                align="start"
                                            >
                                                <Calendar
                                                    v-model="calendarDate"
                                                    initial-focus
                                                />
                                            </PopoverContent>
                                        </Popover>
                                    </div>
                                    <div v-if="isEditMode" class="grid gap-2">
                                        <Label>Status</Label>
                                        <Select v-model="form.is_completed">
                                            <SelectTrigger class="w-full">
                                                <SelectValue
                                                    placeholder="Select status"
                                                />
                                            </SelectTrigger>
                                            <SelectContent>
                                                <SelectItem :value="false"
                                                    >Pending</SelectItem
                                                >
                                                <SelectItem :value="true"
                                                    >Completed</SelectItem
                                                >
                                            </SelectContent>
                                        </Select>
                                    </div>
                                </div>
                            </div>

                            <DialogFooter>
                                <Button
                                    type="button"
                                    variant="outline"
                                    @click="closeModal"
                                    >Cancel</Button
                                >
                                <Button type="submit">
                                    {{
                                        isEditMode ? 'Update Task' : 'Save Task'
                                    }}
                                </Button>
                            </DialogFooter>
                        </form>
                    </DialogContent>
                </Dialog>
            </div>
            <Accordion
                type="multiple"
                class="w-full"
                :default-value="['pending', 'completed']"
            >
                <AccordionItem value="pending">
                    <AccordionTrigger
                        class="text-lg font-semibold text-red-600"
                    >
                        Pending Tasks ({{ pendingTasks.length }})
                    </AccordionTrigger>
                    <AccordionContent>
                        <div class="mt-2 space-y-3">
                            <div
                                v-for="task in pendingTasks"
                                :key="task.id"
                                class="flex items-center justify-between rounded-lg border bg-gray-50 p-4"
                            >
                                <div class="flex flex-col gap-1">
                                    <div class="flex items-center gap-3">
                                        <input
                                            type="checkbox"
                                            :checked="task.is_completed"
                                            @click="toggleTask($event, task)"
                                            class="h-4 w-4 rounded border-gray-300"
                                        />
                                        <span
                                            class="font-semibold text-gray-800"
                                        >
                                            {{ task.title }}
                                        </span>
                                    </div>

                                    <div class="ml-7">
                                        <p
                                            v-if="task.description"
                                            class="text-sm text-gray-600"
                                        >
                                            {{ task.description }}
                                        </p>
                                        <p
                                            v-if="task.due_date"
                                            class="mt-1 text-xs text-gray-400"
                                        >
                                            Due Date: {{ task.due_date }}
                                        </p>
                                    </div>
                                </div>

                                <div class="flex gap-3">
                                    <button
                                        @click="openEditModal(task)"
                                        class="text-blue-600 hover:text-blue-800"
                                    >
                                        <Pencil class="h-5 w-5" />
                                    </button>
                                    <button
                                        @click="deleteTask(task)"
                                        class="text-red-600 hover:text-red-800"
                                    >
                                        <Trash2 class="h-5 w-5" />
                                    </button>
                                </div>
                            </div>
                            <div
                                v-if="pendingTasks.length === 0"
                                class="py-2 text-sm text-gray-500"
                            >
                                No Pending Tasks
                            </div>
                        </div>
                    </AccordionContent>
                </AccordionItem>

                <AccordionItem value="completed">
                    <AccordionTrigger
                        class="text-lg font-semibold text-green-600"
                    >
                        Completed Tasks ({{ completedTasks.length }})
                    </AccordionTrigger>
                    <AccordionContent>
                        <div class="mt-2 space-y-3">
                            <div
                                v-for="task in completedTasks"
                                :key="task.id"
                                class="flex items-center justify-between rounded-lg border bg-green-50 p-4"
                            >
                                <div class="flex flex-col gap-1">
                                    <div class="flex items-center gap-3">
                                        <input
                                            type="checkbox"
                                            :checked="task.is_completed"
                                            @click="toggleTask($event, task)"
                                            class="h-4 w-4 rounded border-gray-300"
                                        />
                                        <span
                                            class="text-gray-500 line-through"
                                        >
                                            {{ task.title }}
                                        </span>
                                    </div>

                                    <div class="ml-7">
                                        <p
                                            v-if="task.description"
                                            class="text-sm text-gray-400 line-through"
                                        >
                                            {{ task.description }}
                                        </p>
                                        <p
                                            v-if="task.due_date"
                                            class="mt-1 text-xs text-gray-400"
                                        >
                                            Completed
                                        </p>
                                    </div>
                                </div>

                                <button
                                    @click="deleteTask(task)"
                                    class="text-red-600 hover:text-red-800"
                                >
                                    <Trash2 class="h-5 w-5" />
                                </button>
                            </div>
                        </div>
                    </AccordionContent>
                </AccordionItem>
            </Accordion>
        </div>
    </div>
</template>
