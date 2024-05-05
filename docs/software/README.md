# Реалізація інформаційного та програмного забезпечення
## Базовий клас з методами присутніми для управління даними таблиці з тасками
```js
const db = require("../database");

class Task {
  static async getAll() {
    const sql = `
        SELECT * 
        FROM tasks;
        `;
    let connection;
    try {
      connection = await db.promisePool.getConnection();
      const tasks = await connection.execute(sql);
      return tasks[0];
    } catch (err) {
      console.error(err);
    } finally {
      if (connection) {
        await connection.release();
      }
    }
  }

  static async getTaskByID(id) {
    const sql = `SELECT * FROM tasks WHERE id = ${id};`;
    let connection;
    try {
      connection = await db.promisePool.getConnection();
      const [res] = await connection.query(sql);
      return res;
    } catch (err) {
      console.error(err);
    } finally {
      if (connection) {
        await connection.release();
      }
    }
  }
  
  static async updateTaskById(id,{ name, developer, status, deadline, projectId}) {
    const sql = `
        UPDATE DB-lab-IO-26.tasks
        SET 
          Name = ?,
          Developer = ?,
          Status = ?,
          Deadline = ?,
		  ProjectId = ?,
        WHERE Id = ?;
    `;

    let connection;
    try {
      connection = await db.promisePool.getConnection();
      await connection.beginTransaction();
      await connection.execute(sql, [
        name,
        developer,
        status,
        deadline,
        projectId,
        id,
      ]);
      await connection.commit();
      return Task.getTaskByID(id);
    } catch (err) {
      await connection.rollback();
      console.log(err);
    } finally {
      if (connection) {
        await connection.release();
      }
    }
  }

  static async createTask({ name, developer, status, deadline, projectId}) {
    const sql = `
        INSERT INTO DB-lab-IO-26.tasks (name, developer, status, deadline, projectId)
        VALUES ( ?, ?, ?, ?, ?);
        `;
    let connection;
    try {
      connection = await db.promisePool.getConnection();
      const [res] = await connection.execute(sql, [
        name,
        developer,
        status,
        deadline,
		projectId,
      ]);
      const createdTaskId = res.insertId;
      console.log(`Task with ID ${createdTaskId} created successfully.`);
      return Task.getTaskByID(createdTaskId);
    } catch (err) {
      console.error(err);
    } finally {
      if (connection) {
        await connection.release();
      }
    }
  }

  static async deleteTaskById(id) {
    const task = await Task.getTaskByID(id);
    const deleteTaskSql = `DELETE FROM DB-lab-IO-26.tasks WHERE id = ${id}`;

    let connection;

    try {
      connection = await db.promisePool.getConnection();
      await connection.beginTransaction();
      await connection.execute(deleteTaskSql);
      await connection.commit();
      return task;
    } catch (err) {
      console.log(err);
      await connection.rollback();
      throw err;
    } finally {
      if (connection) {
        await connection.release();
      }
    }
  }
}

module.exports = { Task };

```
## Контролер тасков
```js

"use strict";

const { Task } = require("../models/Tasks.js");

const asyncWrapper = (callback) => {
  return async function (req, res) {
    const args = [];
    try {
      if (req.params.id) {
        args.push(req.params.id);
      }
      if (req.body) {
        args.push(req.body);
      }
      const result = await callback(...args);
      if (result.length) {
        res.status(200).json({ result });
      } else {
        res.status(404).json({ errorMessage: "No such task" });
      }
    } catch (err) {
      console.error(err);
      res.status(404).json({ errorMessage: err.message });
    }
  };
};

const getAllTasks = asyncWrapper(Task.getAll);
const getTaskById = asyncWrapper(Task.getTaskByID);
const createTask = asyncWrapper(Task.createTask);
const updateTask = asyncWrapper(Task.updateTaskById);
const deleteTask = asyncWrapper(Task.deleteTaskById);

module.exports = {
  getAllTasks,
  getTaskById,
  createTask,
  updateTask,
  deleteTask,
};

```
## CRUD для тасков
```js

"use strict";

const express = require("express");
const TaskController = require("../controllers/TaskController");
const router = new express.Router();
router
  .route("/")
  .get(TaskController.getAllTasks)
  .post(TaskController.createTask);

router
  .route("/:id")
  .get(TaskController.getTaskById)
  .patch(TaskController.updateTask)
  .delete(TaskController.deleteTask);

module.exports = router;

```

