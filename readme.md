// ============================================================
// CommunityHub - Single File Server (Week 11)
// ============================================================
// SETUP:
//   npm install express mongoose bcryptjs jsonwebtoken dotenv
//
// Create a .env file with:
//   MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/community-hub
//   JWT_SECRET=your-super-secret-key
//   JWT_EXPIRES_IN=7d
//   PORT=3000
// ============================================================

require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

// ─────────────────────────────────────────
// DATABASE CONNECTION
// ─────────────────────────────────────────
const connectDB = async () => {
    try {
        const conn = await mongoose.connect(process.env.MONGODB_URI);
        console.log(`✅ MongoDB Connected: ${conn.connection.host}`);
    } catch (error) {
        console.error('❌ MongoDB connection error:', error.message);
        process.exit(1);
    }
};

// ─────────────────────────────────────────
// MODELS
// ─────────────────────────────────────────

// User Model
const userSchema = new mongoose.Schema({
    username: {
        type: String, required: [true, 'Username is required'],
        unique: true, trim: true,
        minlength: [3, 'Username must be at least 3 characters'],
        maxlength: [30, 'Username cannot exceed 30 characters']
    },
    email: {
        type: String, required: [true, 'Email is required'],
        unique: true, lowercase: true,
        match: [/^\S+@\S+\.\S+$/, 'Please enter a valid email']
    },
    password: {
        type: String, required: [true, 'Password is required'],
        minlength: [6, 'Password must be at least 6 characters'],
        select: false
    },
    role: { type: String, enum: ['user', 'admin'], default: 'user' }
}, { timestamps: true });

userSchema.pre('save', async function (next) {
    if (!this.isModified('password')) return next();
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
});

userSchema.methods.comparePassword = async function (candidatePassword) {
    return await bcrypt.compare(candidatePassword, this.password);
};

const User = mongoose.model('User', userSchema);

// Post Model
const postSchema = new mongoose.Schema({
    title: {
        type: String, required: [true, 'Title is required'], trim: true,
        minlength: [3, 'Title must be at least 3 characters'],
        maxlength: [200, 'Title cannot exceed 200 characters']
    },
    content: {
        type: String, required: [true, 'Content is required'],
        minlength: [10, 'Content must be at least 10 characters']
    },
    author: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    likes: { type: Number, default: 0 },
    tags: [{ type: String, trim: true }],
    published: { type: Boolean, default: true }
}, { timestamps: true });

postSchema.index({ title: 'text', content: 'text' });

postSchema.methods.like = function () {
    this.likes++;
    return this.save();
};

postSchema.statics.findByAuthor = function (author) {
    return this.find({ author: new RegExp(author, 'i') });
};

const Post = mongoose.model('Post', postSchema);

// Comment Model
const commentSchema = new mongoose.Schema({
    content: {
        type: String, required: [true, 'Comment content is required'],
        maxlength: [500, 'Comment cannot exceed 500 characters']
    },
    author: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    post: { type: mongoose.Schema.Types.ObjectId, ref: 'Post', required: true }
}, { timestamps: true });

const Comment = mongoose.model('Comment', commentSchema);

// ─────────────────────────────────────────
// MIDDLEWARE
// ─────────────────────────────────────────

const generateToken = (userId) => {
    return jwt.sign({ id: userId }, process.env.JWT_SECRET, {
        expiresIn: process.env.JWT_EXPIRES_IN || '7d'
    });
};

const protect = async (req, res, next) => {
    try {
        const authHeader = req.headers.authorization;
        if (!authHeader || !authHeader.startsWith('Bearer ')) {
            return res.status(401).json({ error: 'Access denied. No token provided.' });
        }
        const token = authHeader.split(' ')[1];
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        const user = await User.findById(decoded.id);
        if (!user) return res.status(401).json({ error: 'User no longer exists' });
        req.user = user;
        next();
    } catch (error) {
        if (error.name === 'JsonWebTokenError') return res.status(401).json({ error: 'Invalid token' });
        if (error.name === 'TokenExpiredError') return res.status(401).json({ error: 'Token expired' });
        next(error);
    }
};

const restrictTo = (...roles) => (req, res, next) => {
    if (!roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'You do not have permission to perform this action' });
    }
    next();
};

// ─────────────────────────────────────────
// AUTH ROUTES
// ─────────────────────────────────────────

// POST /api/auth/register
app.post('/api/auth/register', async (req, res, next) => {
    try {
        const { username, email, password } = req.body;
        const existingUser = await User.findOne({ $or: [{ email }, { username }] });
        if (existingUser) {
            return res.status(400).json({ error: 'User with this email or username already exists' });
        }
        const user = new User({ username, email, password });
        await user.save();
        const token = generateToken(user._id);
        res.status(201).json({
            message: 'User registered successfully', token,
            user: { id: user._id, username: user.username, email: user.email }
        });
    } catch (error) {
        if (error.name === 'ValidationError') {
            const messages = Object.values(error.errors).map(e => e.message);
            return res.status(400).json({ errors: messages });
        }
        next(error);
    }
});

// POST /api/auth/login
app.post('/api/auth/login', async (req, res, next) => {
    try {
        const { email, password } = req.body;
        if (!email || !password) return res.status(400).json({ error: 'Please provide email and password' });
        const user = await User.findOne({ email }).select('+password');
        if (!user) return res.status(401).json({ error: 'Invalid credentials' });
        const isMatch = await user.comparePassword(password);
        if (!isMatch) return res.status(401).json({ error: 'Invalid credentials' });
        const token = generateToken(user._id);
        res.json({
            message: 'Login successful', token,
            user: { id: user._id, username: user.username, email: user.email }
        });
    } catch (error) { next(error); }
});

// GET /api/auth/me
app.get('/api/auth/me', protect, async (req, res, next) => {
    try {
        const user = await User.findById(req.user.id);
        res.json({ id: user._id, username: user.username, email: user.email, role: user.role });
    } catch (error) { next(error); }
});

// ─────────────────────────────────────────
// POST ROUTES
// ─────────────────────────────────────────

// GET /api/posts
app.get('/api/posts', async (req, res, next) => {
    try {
        const { search, sort, page = 1, limit = 10 } = req.query;
        let query = {};
        if (search) query.$text = { $search: search };
        let sortOption = { createdAt: -1 };
        if (sort === 'oldest') sortOption = { createdAt: 1 };
        else if (sort === 'popular') sortOption = { likes: -1 };
        const skip = (page - 1) * limit;
        const posts = await Post.find(query)
            .populate('author', 'username')
            .sort(sortOption).skip(skip).limit(parseInt(limit));
        const total = await Post.countDocuments(query);
        res.json({ posts, pagination: { page: parseInt(page), limit: parseInt(limit), total, pages: Math.ceil(total / limit) } });
    } catch (error) { next(error); }
});

// GET /api/posts/:id
app.get('/api/posts/:id', async (req, res, next) => {
    try {
        const post = await Post.findById(req.params.id).populate('author', 'username');
        if (!post) return res.status(404).json({ error: 'Post not found' });
        res.json(post);
    } catch (error) {
        if (error.name === 'CastError') return res.status(400).json({ error: 'Invalid post ID' });
        next(error);
    }
});

// POST /api/posts
app.post('/api/posts', protect, async (req, res, next) => {
    try {
        const { title, content, tags } = req.body;
        const post = new Post({ title, content, author: req.user._id, tags });
        await post.save();
        await post.populate('author', 'username email');
        res.status(201).json(post);
    } catch (error) {
        if (error.name === 'ValidationError') {
            const messages = Object.values(error.errors).map(e => e.message);
            return res.status(400).json({ errors: messages });
        }
        next(error);
    }
});

// PUT /api/posts/:id
app.put('/api/posts/:id', protect, async (req, res, next) => {
    try {
        const post = await Post.findById(req.params.id);
        if (!post) return res.status(404).json({ error: 'Post not found' });
        if (post.author.toString() !== req.user._id.toString()) {
            return res.status(403).json({ error: 'You can only edit your own posts' });
        }
        const { title, content, tags } = req.body;
        post.title = title || post.title;
        post.content = content || post.content;
        post.tags = tags || post.tags;
        await post.save();
        res.json(post);
    } catch (error) { next(error); }
});

// DELETE /api/posts/:id
app.delete('/api/posts/:id', protect, async (req, res, next) => {
    try {
        const post = await Post.findById(req.params.id);
        if (!post) return res.status(404).json({ error: 'Post not found' });
        if (post.author.toString() !== req.user._id.toString() && req.user.role !== 'admin') {
            return res.status(403).json({ error: 'You can only delete your own posts' });
        }
        await Post.findByIdAndDelete(req.params.id);
        res.status(204).send();
    } catch (error) { next(error); }
});

// POST /api/posts/:id/like
app.post('/api/posts/:id/like', async (req, res, next) => {
    try {
        const post = await Post.findById(req.params.id);
        if (!post) return res.status(404).json({ error: 'Post not found' });
        await post.like();
        res.json(post);
    } catch (error) { next(error); }
});

// ─────────────────────────────────────────
// COMMENT ROUTES
// ─────────────────────────────────────────

// GET /api/posts/:postId/comments
app.get('/api/posts/:postId/comments', async (req, res, next) => {
    try {
        const comments = await Comment.find({ post: req.params.postId })
            .populate('author', 'username')
            .sort({ createdAt: -1 });
        res.json(comments);
    } catch (error) { next(error); }
});

// POST /api/posts/:postId/comments
app.post('/api/posts/:postId/comments', protect, async (req, res, next) => {
    try {
        const { content } = req.body;
        const post = await Post.findById(req.params.postId);
        if (!post) return res.status(404).json({ error: 'Post not found' });
        const comment = new Comment({ content, author: req.user._id, post: req.params.postId });
        await comment.save();
        await comment.populate('author', 'username');
        res.status(201).json(comment);
    } catch (error) { next(error); }
});

// DELETE /api/posts/:postId/comments/:commentId
app.delete('/api/posts/:postId/comments/:commentId', protect, async (req, res, next) => {
    try {
        const comment = await Comment.findById(req.params.commentId);
        if (!comment) return res.status(404).json({ error: 'Comment not found' });
        if (comment.author.toString() !== req.user._id.toString() && req.user.role !== 'admin') {
            return res.status(403).json({ error: 'You can only delete your own comments' });
        }
        await Comment.findByIdAndDelete(req.params.commentId);
        res.status(204).send();
    } catch (error) { next(error); }
});

// ─────────────────────────────────────────
// USER ROUTES
// ─────────────────────────────────────────

// PUT /api/users/me
app.put('/api/users/me', protect, async (req, res, next) => {
    try {
        const { username, email } = req.body;
        const user = await User.findByIdAndUpdate(
            req.user._id, { username, email }, { new: true, runValidators: true }
        );
        res.json({ id: user._id, username: user.username, email: user.email });
    } catch (error) { next(error); }
});

// GET /api/users/:id/posts
app.get('/api/users/:id/posts', async (req, res, next) => {
    try {
        const posts = await Post.find({ author: req.params.id })
            .populate('author', 'username')
            .sort({ createdAt: -1 });
        res.json(posts);
    } catch (error) { next(error); }
});

// ─────────────────────────────────────────
// FRONTEND (HTML)
// ─────────────────────────────────────────
app.get('/', (req, res) => {
    res.send(`<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>CommunityHub</title>
<link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:ital,wght@0,300;0,400;0,500;1,300&display=swap" rel="stylesheet">
<style>
  :root {
    --bg: #0a0a0f;
    --surface: #13131a;
    --surface2: #1c1c28;
    --border: #2a2a3d;
    --accent: #7c6af5;
    --accent2: #f5a56a;
    --text: #e8e8f0;
    --muted: #6b6b8a;
    --danger: #f56a6a;
    --success: #6af5a5;
    --radius: 12px;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { background: var(--bg); color: var(--text); font-family: 'DM Sans', sans-serif; min-height: 100vh; }

  /* NOISE TEXTURE */
  body::before {
    content: '';
    position: fixed; inset: 0; pointer-events: none; z-index: 0;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.035'/%3E%3C/svg%3E");
    opacity: 0.4;
  }

  /* LAYOUT */
  header {
    position: sticky; top: 0; z-index: 100;
    background: rgba(10,10,15,0.85); backdrop-filter: blur(16px);
    border-bottom: 1px solid var(--border);
    padding: 0 2rem;
    display: flex; align-items: center; justify-content: space-between;
    height: 64px;
  }
  .logo {
    font-family: 'Syne', sans-serif; font-weight: 800; font-size: 1.3rem;
    background: linear-gradient(135deg, var(--accent), var(--accent2));
    -webkit-background-clip: text; -webkit-text-fill-color: transparent;
    letter-spacing: -0.02em;
  }
  .header-right { display: flex; align-items: center; gap: 1rem; }
  #userInfo { font-size: 0.85rem; color: var(--muted); }
  #userInfo span { color: var(--accent2); font-weight: 500; }

  .container { max-width: 1100px; margin: 0 auto; padding: 2.5rem 2rem; position: relative; z-index: 1; }

  /* TABS */
  .tabs { display: flex; gap: 0.25rem; margin-bottom: 2.5rem; background: var(--surface); padding: 0.25rem; border-radius: 10px; width: fit-content; }
  .tab {
    padding: 0.5rem 1.25rem; border-radius: 8px; border: none; background: transparent;
    color: var(--muted); font-family: 'DM Sans', sans-serif; font-size: 0.9rem; cursor: pointer;
    transition: all 0.2s;
  }
  .tab.active { background: var(--surface2); color: var(--text); box-shadow: 0 0 0 1px var(--border); }
  .tab:hover:not(.active) { color: var(--text); }

  .panel { display: none; }
  .panel.active { display: block; animation: fadeUp 0.3s ease; }
  @keyframes fadeUp { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }

  /* CARDS */
  .card {
    background: var(--surface); border: 1px solid var(--border);
    border-radius: var(--radius); padding: 1.75rem;
    transition: border-color 0.2s;
  }
  .card:hover { border-color: rgba(124,106,245,0.3); }

  .card-title {
    font-family: 'Syne', sans-serif; font-weight: 700; font-size: 1.1rem;
    margin-bottom: 1.25rem; color: var(--text);
  }

  /* FORM */
  .form-group { margin-bottom: 1rem; }
  label { display: block; font-size: 0.8rem; font-weight: 500; color: var(--muted); margin-bottom: 0.4rem; letter-spacing: 0.04em; text-transform: uppercase; }
  input, textarea {
    width: 100%; background: var(--surface2); border: 1px solid var(--border);
    border-radius: 8px; padding: 0.65rem 0.9rem; color: var(--text);
    font-family: 'DM Sans', sans-serif; font-size: 0.95rem;
    transition: border-color 0.2s, box-shadow 0.2s; outline: none;
  }
  input:focus, textarea:focus { border-color: var(--accent); box-shadow: 0 0 0 3px rgba(124,106,245,0.15); }
  textarea { resize: vertical; min-height: 100px; }

  /* BUTTONS */
  .btn {
    padding: 0.6rem 1.3rem; border-radius: 8px; border: none; cursor: pointer;
    font-family: 'DM Sans', sans-serif; font-size: 0.9rem; font-weight: 500;
    transition: all 0.18s; display: inline-flex; align-items: center; gap: 0.4rem;
  }
  .btn-primary { background: var(--accent); color: #fff; }
  .btn-primary:hover { background: #6a58e0; transform: translateY(-1px); box-shadow: 0 4px 12px rgba(124,106,245,0.35); }
  .btn-ghost { background: transparent; color: var(--muted); border: 1px solid var(--border); }
  .btn-ghost:hover { color: var(--text); border-color: var(--accent); }
  .btn-danger { background: transparent; color: var(--danger); border: 1px solid rgba(245,106,106,0.3); font-size: 0.8rem; padding: 0.3rem 0.7rem; }
  .btn-danger:hover { background: rgba(245,106,106,0.1); }
  .btn-sm { padding: 0.35rem 0.8rem; font-size: 0.82rem; }
  .btn:disabled { opacity: 0.5; cursor: not-allowed; transform: none !important; }

  /* GRID */
  .two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem; }
  @media(max-width: 680px) { .two-col { grid-template-columns: 1fr; } }

  /* POSTS */
  .posts-grid { display: flex; flex-direction: column; gap: 1rem; }
  .post-card {
    background: var(--surface); border: 1px solid var(--border);
    border-radius: var(--radius); padding: 1.5rem;
    transition: border-color 0.2s, transform 0.2s;
  }
  .post-card:hover { border-color: rgba(124,106,245,0.4); transform: translateX(3px); }
  .post-card-title {
    font-family: 'Syne', sans-serif; font-weight: 700; font-size: 1.05rem;
    color: var(--text); cursor: pointer; margin-bottom: 0.5rem;
  }
  .post-card-title:hover { color: var(--accent); }
  .post-card-content { color: var(--muted); font-size: 0.9rem; line-height: 1.6; margin-bottom: 1rem; }
  .post-meta { display: flex; align-items: center; gap: 1rem; font-size: 0.8rem; color: var(--muted); flex-wrap: wrap; }
  .post-meta .author { color: var(--accent2); }
  .post-meta .date { }
  .tags { display: flex; gap: 0.4rem; flex-wrap: wrap; margin-top: 0.75rem; }
  .tag {
    background: rgba(124,106,245,0.12); color: var(--accent);
    border: 1px solid rgba(124,106,245,0.2);
    border-radius: 20px; padding: 0.2rem 0.65rem; font-size: 0.75rem;
  }
  .like-btn {
    background: transparent; border: 1px solid var(--border); color: var(--muted);
    border-radius: 6px; padding: 0.3rem 0.7rem; cursor: pointer; font-size: 0.82rem;
    transition: all 0.18s; display: inline-flex; align-items: center; gap: 0.3rem;
  }
  .like-btn:hover { border-color: var(--accent2); color: var(--accent2); }
  .post-actions { display: flex; gap: 0.5rem; align-items: center; }

  /* ALERTS */
  .alert {
    padding: 0.85rem 1rem; border-radius: 8px; font-size: 0.88rem;
    margin-bottom: 1rem; display: none;
    animation: fadeUp 0.2s ease;
  }
  .alert.show { display: block; }
  .alert-error { background: rgba(245,106,106,0.1); border: 1px solid rgba(245,106,106,0.25); color: #f99; }
  .alert-success { background: rgba(106,245,165,0.1); border: 1px solid rgba(106,245,165,0.25); color: var(--success); }

  /* MODAL */
  .modal-overlay {
    position: fixed; inset: 0; background: rgba(0,0,0,0.7); backdrop-filter: blur(4px);
    z-index: 200; display: none; align-items: center; justify-content: center; padding: 1rem;
  }
  .modal-overlay.open { display: flex; }
  .modal {
    background: var(--surface); border: 1px solid var(--border);
    border-radius: 16px; padding: 2rem; width: 100%; max-width: 640px;
    max-height: 80vh; overflow-y: auto;
    animation: fadeUp 0.25s ease;
  }
  .modal-title { font-family: 'Syne', sans-serif; font-weight: 700; font-size: 1.2rem; margin-bottom: 1.5rem; }
  .modal-close { float: right; background: none; border: none; color: var(--muted); cursor: pointer; font-size: 1.3rem; margin-top: -4px; }

  /* COMMENTS */
  .comment-item {
    border-top: 1px solid var(--border); padding-top: 1rem; margin-top: 1rem;
  }
  .comment-author { font-size: 0.82rem; color: var(--accent2); font-weight: 500; }
  .comment-date { font-size: 0.78rem; color: var(--muted); margin-left: 0.5rem; }
  .comment-content { font-size: 0.9rem; color: var(--muted); margin-top: 0.3rem; line-height: 1.5; }

  /* DIVIDER */
  .divider { border: none; border-top: 1px solid var(--border); margin: 1.5rem 0; }

  /* EMPTY */
  .empty { text-align: center; padding: 3rem 1rem; color: var(--muted); }
  .empty-icon { font-size: 2.5rem; margin-bottom: 0.75rem; opacity: 0.4; }

  /* LOADER */
  .loader { display: flex; justify-content: center; padding: 2rem; }
  .spinner {
    width: 28px; height: 28px; border: 2px solid var(--border);
    border-top-color: var(--accent); border-radius: 50%;
    animation: spin 0.7s linear infinite;
  }
  @keyframes spin { to { transform: rotate(360deg); } }

  /* SEARCH BAR */
  .search-row { display: flex; gap: 0.75rem; margin-bottom: 1.5rem; }
  .search-row input { flex: 1; }

  .sort-select {
    background: var(--surface2); border: 1px solid var(--border); color: var(--muted);
    border-radius: 8px; padding: 0.65rem 0.9rem;
    font-family: 'DM Sans', sans-serif; font-size: 0.9rem; cursor: pointer; outline: none;
  }
  .sort-select:focus { border-color: var(--accent); }

  /* WELCOME BANNER */
  .banner {
    background: linear-gradient(135deg, rgba(124,106,245,0.12), rgba(245,165,106,0.08));
    border: 1px solid rgba(124,106,245,0.2);
    border-radius: var(--radius); padding: 1.75rem 2rem;
    margin-bottom: 2rem;
  }
  .banner h1 { font-family: 'Syne', sans-serif; font-weight: 800; font-size: 1.8rem; margin-bottom: 0.4rem; }
  .banner p { color: var(--muted); font-size: 0.95rem; }
</style>
</head>
<body>

<header>
  <div class="logo">◈ CommunityHub</div>
  <div class="header-right">
    <div id="userInfo"></div>
    <button class="btn btn-ghost btn-sm" id="authToggleBtn" onclick="toggleAuthPanel()">Sign In</button>
    <button class="btn btn-danger btn-sm" id="logoutBtn" style="display:none" onclick="logout()">Sign Out</button>
  </div>
</header>

<div class="container">

  <!-- BANNER -->
  <div class="banner" id="welcomeBanner">
    <h1>Welcome to CommunityHub 🔐</h1>
    <p>Sign in or create an account to start posting and joining the conversation.</p>
  </div>

  <!-- TABS -->
  <div class="tabs">
    <button class="tab active" onclick="showTab('posts')">Posts</button>
    <button class="tab" onclick="showTab('auth')" id="authTab">Account</button>
    <button class="tab" onclick="showTab('newPost')" id="newPostTab" style="display:none">New Post</button>
  </div>

  <!-- POSTS PANEL -->
  <div class="panel active" id="postsPanel">
    <div class="search-row">
      <input id="searchInput" placeholder="Search posts…" oninput="debounceSearch()" />
      <select class="sort-select" id="sortSelect" onchange="loadPosts()">
        <option value="">Newest</option>
        <option value="oldest">Oldest</option>
        <option value="popular">Most Liked</option>
      </select>
    </div>
    <div id="postsList"></div>
  </div>

  <!-- AUTH PANEL -->
  <div class="panel" id="authPanel">
    <div class="two-col">
      <!-- LOGIN -->
      <div class="card">
        <div class="card-title">Sign In</div>
        <div class="alert" id="loginAlert"></div>
        <div class="form-group"><label>Email</label><input id="loginEmail" type="email" placeholder="you@example.com" /></div>
        <div class="form-group"><label>Password</label><input id="loginPassword" type="password" placeholder="••••••" /></div>
        <button class="btn btn-primary" onclick="login()" style="width:100%">Sign In</button>
      </div>
      <!-- REGISTER -->
      <div class="card">
        <div class="card-title">Create Account</div>
        <div class="alert" id="registerAlert"></div>
        <div class="form-group"><label>Username</label><input id="regUsername" placeholder="cooluser" /></div>
        <div class="form-group"><label>Email</label><input id="regEmail" type="email" placeholder="you@example.com" /></div>
        <div class="form-group"><label>Password</label><input id="regPassword" type="password" placeholder="min 6 characters" /></div>
        <button class="btn btn-primary" onclick="register()" style="width:100%">Create Account</button>
      </div>
    </div>
  </div>

  <!-- NEW POST PANEL -->
  <div class="panel" id="newPostPanel">
    <div class="card" style="max-width: 680px">
      <div class="card-title">Create New Post</div>
      <div class="alert" id="postAlert"></div>
      <div class="form-group"><label>Title</label><input id="postTitle" placeholder="What's on your mind?" /></div>
      <div class="form-group"><label>Content</label><textarea id="postContent" placeholder="Write something interesting…"></textarea></div>
      <div class="form-group"><label>Tags <span style="color:var(--muted);font-size:0.75rem;text-transform:none">(comma separated)</span></label><input id="postTags" placeholder="tech, news, community" /></div>
      <div style="display:flex;gap:0.75rem;margin-top:0.5rem">
        <button class="btn btn-primary" onclick="createPost()">Publish Post</button>
        <button class="btn btn-ghost" onclick="showTab('posts')">Cancel</button>
      </div>
    </div>
  </div>

</div>

<!-- POST MODAL -->
<div class="modal-overlay" id="postModal">
  <div class="modal">
    <button class="modal-close" onclick="closeModal()">✕</button>
    <div id="modalContent"></div>
  </div>
</div>

<script>
  let token = localStorage.getItem('chToken');
  let currentUser = JSON.parse(localStorage.getItem('chUser') || 'null');
  let searchTimeout = null;

  const API = '';

  // ── INIT ──
  updateAuthUI();
  loadPosts();

  function updateAuthUI() {
    const userInfo = document.getElementById('userInfo');
    const authToggleBtn = document.getElementById('authToggleBtn');
    const logoutBtn = document.getElementById('logoutBtn');
    const newPostTab = document.getElementById('newPostTab');
    const welcomeBanner = document.getElementById('welcomeBanner');

    if (currentUser) {
      userInfo.innerHTML = 'Hello, <span>' + currentUser.username + '</span>';
      authToggleBtn.style.display = 'none';
      logoutBtn.style.display = '';
      newPostTab.style.display = '';
      welcomeBanner.style.display = 'none';
    } else {
      userInfo.innerHTML = '';
      authToggleBtn.style.display = '';
      logoutBtn.style.display = 'none';
      newPostTab.style.display = 'none';
      welcomeBanner.style.display = '';
    }
  }

  function showTab(tab) {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
    const tabMap = { posts: 0, auth: 1, newPost: 2 };
    document.querySelectorAll('.tab')[tabMap[tab] !== undefined ? tabMap[tab] : 0].classList.add('active');
    const panels = { posts: 'postsPanel', auth: 'authPanel', newPost: 'newPostPanel' };
    document.getElementById(panels[tab]).classList.add('active');
  }

  function toggleAuthPanel() { showTab('auth'); }

  function logout() {
    token = null; currentUser = null;
    localStorage.removeItem('chToken'); localStorage.removeItem('chUser');
    updateAuthUI(); showTab('posts');
  }

  // ── AUTH ──
  async function login() {
    const email = document.getElementById('loginEmail').value;
    const password = document.getElementById('loginPassword').value;
    try {
      const res = await fetch(API + '/api/auth/login', {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      });
      const data = await res.json();
      if (!res.ok) { showAlert('loginAlert', data.error || 'Login failed', 'error'); return; }
      token = data.token; currentUser = data.user;
      localStorage.setItem('chToken', token);
      localStorage.setItem('chUser', JSON.stringify(currentUser));
      updateAuthUI(); showTab('posts'); showAlert('loginAlert', 'Welcome back, ' + currentUser.username + '!', 'success');
    } catch (e) { showAlert('loginAlert', 'Connection error', 'error'); }
  }

  async function register() {
    const username = document.getElementById('regUsername').value;
    const email = document.getElementById('regEmail').value;
    const password = document.getElementById('regPassword').value;
    try {
      const res = await fetch(API + '/api/auth/register', {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username, email, password })
      });
      const data = await res.json();
      if (!res.ok) {
        const msg = data.errors ? data.errors.join(', ') : data.error;
        showAlert('registerAlert', msg, 'error'); return;
      }
      token = data.token; currentUser = data.user;
      localStorage.setItem('chToken', token);
      localStorage.setItem('chUser', JSON.stringify(currentUser));
      updateAuthUI(); showTab('posts'); showAlert('registerAlert', 'Account created! Welcome, ' + currentUser.username, 'success');
    } catch (e) { showAlert('registerAlert', 'Connection error', 'error'); }
  }

  // ── POSTS ──
  async function loadPosts() {
    const el = document.getElementById('postsList');
    el.innerHTML = '<div class="loader"><div class="spinner"></div></div>';
    const search = document.getElementById('searchInput').value;
    const sort = document.getElementById('sortSelect').value;
    let url = API + '/api/posts?limit=20';
    if (search) url += '&search=' + encodeURIComponent(search);
    if (sort) url += '&sort=' + sort;
    try {
      const res = await fetch(url);
      const data = await res.json();
      if (!data.posts || data.posts.length === 0) {
        el.innerHTML = '<div class="empty"><div class="empty-icon">✦</div><div>No posts yet. Be the first!</div></div>';
        return;
      }
      el.innerHTML = '<div class="posts-grid">' + data.posts.map(renderPost).join('') + '</div>';
    } catch (e) { el.innerHTML = '<div class="empty"><div class="empty-icon">⚠</div><div>Could not load posts.</div></div>'; }
  }

  function renderPost(p) {
    const isOwner = currentUser && p.author && (p.author._id === currentUser.id || p.author === currentUser.id);
    const authorName = p.author ? (p.author.username || p.author) : 'Unknown';
    const dateStr = new Date(p.createdAt).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
    const preview = p.content.length > 140 ? p.content.slice(0, 140) + '…' : p.content;
    const tagsHtml = p.tags && p.tags.length ? '<div class="tags">' + p.tags.map(t => '<span class="tag">' + t + '</span>').join('') + '</div>' : '';
    const ownerBtns = isOwner ? '<button class="btn btn-danger" onclick="deletePost(event,\'' + p._id + '\')">Delete</button>' : '';
    return \`<div class="post-card" id="post-\${p._id}">
      <div class="post-card-title" onclick="openPost('\${p._id}')">\${escHtml(p.title)}</div>
      <div class="post-card-content">\${escHtml(preview)}</div>
      \${tagsHtml}
      <div style="display:flex;justify-content:space-between;align-items:center;margin-top:1rem;flex-wrap:wrap;gap:0.5rem">
        <div class="post-meta">
          <span class="author">@\${escHtml(authorName)}</span>
          <span class="date">\${dateStr}</span>
        </div>
        <div class="post-actions">
          <button class="like-btn" onclick="likePost(event,'\${p._id}',this)">♥ \${p.likes}</button>
          \${ownerBtns}
        </div>
      </div>
    </div>\`;
  }

  async function createPost() {
    const title = document.getElementById('postTitle').value;
    const content = document.getElementById('postContent').value;
    const tags = document.getElementById('postTags').value.split(',').map(t => t.trim()).filter(Boolean);
    try {
      const res = await fetch(API + '/api/posts', {
        method: 'POST', headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + token },
        body: JSON.stringify({ title, content, tags })
      });
      const data = await res.json();
      if (!res.ok) {
        const msg = data.errors ? data.errors.join(', ') : data.error;
        showAlert('postAlert', msg, 'error'); return;
      }
      document.getElementById('postTitle').value = '';
      document.getElementById('postContent').value = '';
      document.getElementById('postTags').value = '';
      showAlert('postAlert', 'Post published!', 'success');
      setTimeout(() => { showTab('posts'); loadPosts(); }, 800);
    } catch (e) { showAlert('postAlert', 'Connection error', 'error'); }
  }

  async function likePost(e, id, btn) {
    e.stopPropagation();
    try {
      const res = await fetch(API + '/api/posts/' + id + '/like', { method: 'POST' });
      const data = await res.json();
      btn.textContent = '♥ ' + data.likes;
    } catch (e) {}
  }

  async function deletePost(e, id) {
    e.stopPropagation();
    if (!confirm('Delete this post?')) return;
    try {
      await fetch(API + '/api/posts/' + id, { method: 'DELETE', headers: { 'Authorization': 'Bearer ' + token } });
      document.getElementById('post-' + id).remove();
    } catch (e) {}
  }

  // ── MODAL / COMMENTS ──
  async function openPost(id) {
    document.getElementById('postModal').classList.add('open');
    document.getElementById('modalContent').innerHTML = '<div class="loader"><div class="spinner"></div></div>';
    try {
      const [postRes, commentsRes] = await Promise.all([
        fetch(API + '/api/posts/' + id),
        fetch(API + '/api/posts/' + id + '/comments')
      ]);
      const post = await postRes.json();
      const comments = await commentsRes.json();
      const authorName = post.author ? (post.author.username || post.author) : 'Unknown';
      const tagsHtml = post.tags && post.tags.length ? '<div class="tags">' + post.tags.map(t => '<span class="tag">' + t + '</span>').join('') + '</div>' : '';
      const commentsHtml = comments.length ? comments.map(c => {
        const a = c.author ? (c.author.username || c.author) : 'Unknown';
        return \`<div class="comment-item">
          <span class="comment-author">@\${escHtml(a)}</span>
          <span class="comment-date">\${new Date(c.createdAt).toLocaleDateString()}</span>
          <div class="comment-content">\${escHtml(c.content)}</div>
        </div>\`;
      }).join('') : '<div style="color:var(--muted);font-size:0.88rem;padding-top:1rem">No comments yet.</div>';

      const commentForm = currentUser ? \`
        <hr class="divider">
        <div style="font-family:'Syne',sans-serif;font-weight:600;font-size:0.95rem;margin-bottom:0.75rem">Leave a comment</div>
        <textarea id="commentInput" placeholder="Write a comment…" style="margin-bottom:0.75rem"></textarea>
        <button class="btn btn-primary btn-sm" onclick="addComment('\${post._id}')">Post Comment</button>
        <div class="alert" id="commentAlert"></div>
      \` : '<p style="color:var(--muted);font-size:0.88rem;margin-top:1rem">Sign in to comment.</p>';

      document.getElementById('modalContent').innerHTML = \`
        <h2 style="font-family:'Syne',sans-serif;font-weight:700;font-size:1.3rem;margin-bottom:0.75rem">\${escHtml(post.title)}</h2>
        <div style="color:var(--muted);font-size:0.82rem;margin-bottom:1rem">@\${escHtml(authorName)} · \${new Date(post.createdAt).toLocaleDateString()}</div>
        \${tagsHtml}
        <p style="line-height:1.75;color:#c8c8d8;margin-top:1rem">\${escHtml(post.content)}</p>
        <hr class="divider">
        <div style="font-family:'Syne',sans-serif;font-weight:600;font-size:0.95rem;margin-bottom:0.5rem">Comments (\${comments.length})</div>
        <div id="commentsContainer">\${commentsHtml}</div>
        \${commentForm}
      \`;
    } catch (e) { document.getElementById('modalContent').innerHTML = '<p style="color:var(--danger)">Failed to load post.</p>'; }
  }

  async function addComment(postId) {
    const content = document.getElementById('commentInput').value;
    if (!content.trim()) return;
    try {
      const res = await fetch(API + '/api/posts/' + postId + '/comments', {
        method: 'POST', headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + token },
        body: JSON.stringify({ content })
      });
      const data = await res.json();
      if (!res.ok) { showAlert('commentAlert', data.error || 'Error', 'error'); return; }
      const a = data.author ? (data.author.username || data.author) : 'You';
      const html = \`<div class="comment-item">
        <span class="comment-author">@\${escHtml(a)}</span>
        <span class="comment-date">just now</span>
        <div class="comment-content">\${escHtml(data.content)}</div>
      </div>\`;
      document.getElementById('commentsContainer').insertAdjacentHTML('afterbegin', html);
      document.getElementById('commentInput').value = '';
    } catch (e) { showAlert('commentAlert', 'Connection error', 'error'); }
  }

  function closeModal() { document.getElementById('postModal').classList.remove('open'); }
  document.getElementById('postModal').addEventListener('click', function(e) { if (e.target === this) closeModal(); });

  // ── UTILS ──
  function showAlert(id, msg, type) {
    const el = document.getElementById(id);
    el.textContent = msg;
    el.className = 'alert show alert-' + type;
    setTimeout(() => { el.className = 'alert'; }, 4000);
  }

  function escHtml(s) {
    return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
  }

  function debounceSearch() {
    clearTimeout(searchTimeout);
    searchTimeout = setTimeout(loadPosts, 350);
  }
</script>
</body>
</html>`);
});

// ─────────────────────────────────────────
// ERROR HANDLER
// ─────────────────────────────────────────
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ error: 'Something went wrong!', message: err.message });
});

// ─────────────────────────────────────────
// START
// ─────────────────────────────────────────
const PORT = process.env.PORT || 3000;

connectDB().then(() => {
    app.listen(PORT, () => {
        console.log(`🚀 Server running at http://localhost:${PORT}`);
    });
});