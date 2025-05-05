// === server.js ===
require('dotenv').config();
const express = require('express');
const { graphqlHTTP } = require('express-graphql');
const { buildSchema } = require('graphql');
const mongoose = require('mongoose');
const jwt = require('jsonwebtoken');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

// === MongoDB Models ===
const { Schema, model } = mongoose;

const userSchema = new Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
  enrolledCourses: [{ type: Schema.Types.ObjectId, ref: 'Course' }],
  progress: [{
    course: { type: Schema.Types.ObjectId, ref: 'Course' },
    completedLessons: [String]
  }]
});
const User = model('User', userSchema);

const courseSchema = new Schema({
  title: String,
  description: String,
  category: String,
  lessons: [String],
  quizzes: [{
    question: String,
    options: [String],
    answer: String
  }],
  enrolledStudents: [{ type: Schema.Types.ObjectId, ref: 'User' }]
});
const Course = model('Course', courseSchema);

// === Connect to MongoDB Atlas ===
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log("MongoDB connected"))
  .catch(err => console.error("MongoDB error:", err));

// === JWT Middleware ===
const isAuth = (req, res, next) => {
  const authHeader = req.headers.authorization || '';
  const token = authHeader.split(' ')[1];
  if (!token) return next();
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.isAuth = true;
    req.userId = decoded.userId;
  } catch (err) {
    req.isAuth = false;
  }
  next();
};

app.use(isAuth);

// === GraphQL Schema ===
const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    email: String!
    enrolledCourses: [Course]
    progress: [Progress]
  }

  type Course {
    id: ID!
    title: String!
    description: String!
    category: String
    lessons: [String]
    quizzes: [Quiz]
    enrolledStudents: [User]
  }

  type Quiz {
    question: String!
    options: [String]!
    answer: String!
  }

  type Progress {
    course: Course
    completedLessons: [String]
  }

  type AuthData {
    token: String
    userId: ID
  }

  type Query {
    courses(category: String): [Course]
    myCourses: [Course]
    myProgress(courseId: ID!): Progress
  }

  type Mutation {
    register(name: String!, email: String!, password: String!): User
    login(email: String!, password: String!): AuthData
    enroll(courseId: ID!): Course
    completeLesson(courseId: ID!, lesson: String!): Progress
  }
`);

const root = {
  courses: async ({ category }) => {
    return category ? await Course.find({ category }) : await Course.find();
  },

  myCourses: async (args, req) => {
    if (!req.isAuth) throw new Error("Unauthorized");
    const user = await User.findById(req.userId).populate('enrolledCourses');
    return user.enrolledCourses;
  },

  myProgress: async ({ courseId }, req) => {
    if (!req.isAuth) throw new Error("Unauthorized");
    const user = await User.findById(req.userId).populate('progress.course');
    return user.progress.find(p => p.course._id.toString() === courseId);
  },

  register: async ({ name, email, password }) => {
    const existingUser = await User.findOne({ email });
    if (existingUser) throw new Error("Email already registered");
    const user = new User({ name, email, password });
    return await user.save();
  },

  login: async ({ email, password }) => {
    const user = await User.findOne({ email });
    if (!user || user.password !== password) throw new Error("Invalid credentials");
    const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET, { expiresIn: '1d' });
    return { token, userId: user._id };
  },

  enroll: async ({ courseId }, req) => {
    if (!req.isAuth) throw new Error("Unauthorized");
    const user = await User.findById(req.userId);
    const course = await Course.findById(courseId);

    if (!user.enrolledCourses.includes(courseId)) {
      user.enrolledCourses.push(courseId);
      course.enrolledStudents.push(user._id);
      user.progress.push({ course: courseId, completedLessons: [] });
      await user.save();
      await course.save();
    }
    return course;
  },

  completeLesson: async ({ courseId, lesson }, req) => {
    if (!req.isAuth) throw new Error("Unauthorized");
    const user = await User.findById(req.userId);
    const progress = user.progress.find(p => p.course.toString() === courseId);
    if (!progress.completedLessons.includes(lesson)) {
      progress.completedLessons.push(lesson);
    }
    await user.save();
    return progress;
  }
};

app.use('/graphql', graphqlHTTP({
  schema,
  rootValue: root,
  graphiql: true
}));

app.listen(3000, () => console.log("Server is running on http://localhost:3000/graphql"));
